# Initialize an OpenLayers Map in Angular

This lesson investigates the failure modes that emerge when an imperative mapping library is dropped into Angular's component lifecycle without explicit ownership. You will build a MapHostComponent that initializes at the right moment, runs map events outside Angular's zone, and cleans up completely on destruction.

> **Learning Objectives**  
> - Diagnose why naive OpenLayers initialization in Angular causes memory leaks and redundant change detection
> - Explain the boundary between Angular's declarative change detection and OpenLayers' imperative render loop
> - Implement map initialization in ngAfterViewInit using a ViewChild container reference
> - Wrap OpenLayers event listeners in NgZone.runOutsideAngular to prevent unnecessary change detection cycles
> - Tear down map resources explicitly in ngOnDestroy using map.setTarget(null) and source disposal

## When the Map Outlives the Component

Consider a developer who just installed ol in their Angular project and wants to display a tunnel control map. They write something that feels natural — create the map in ngOnInit, target a div with an id, and move on.

``` TypeScript
import { Component, OnInit } from '@angular/core';
import Map from 'ol/Map';
import View from 'ol/View';
import TileLayer from 'ol/layer/Tile';
import OSM from 'ol/source/OSM';

@Component({
  selector: 'app-tunnel-map',
  template: `<div id="map-container" style="height: 100%; width: 100%;"></div>`,
})
export class TunnelMapComponent implements OnInit {
  private map!: Map;

  ngOnInit(): void {
    this.map = new Map({
      target: 'map-container',
      layers: [
        new TileLayer({ source: new OSM() }),
      ],
      view: new View({
        center: [0, 0],
        zoom: 2,
      }),
    });
  }
}
```

The map renders. The developer moves on. But when they navigate to a different route and come back, the DevTools Memory tab tells a different story. The old canvas element is still there — detached from the document, but not garbage collected. Each navigation adds another. After five round trips, there are five maps, five sets of event listeners on the window, and five render loops competing for requestAnimationFrame.

Before reading on, predict: what specifically prevents Angular from cleaning up the OpenLayers map when the component is destroyed? Is it the DOM, the listeners, the Map object itself — or all three?

Take a moment to write down your prediction before the next section reveals the mechanism.

The tempting wrong model here is that OpenLayers map DOM is just like any other Angular template content — it lives inside the component's template, so Angular should destroy it when the component is destroyed.

Test that assumption. Open DevTools, navigate to the map component, then navigate away. Inspect the Elements panel. The div with id map-container is gone — Angular did remove it from the DOM. Now switch to the Memory panel and take a heap snapshot. Search for HTMLCanvasElement. The canvas that OpenLayers created inside that div is still allocated. Why? Because the Map instance holds a direct JavaScript reference to it, and the Map instance itself is referenced by event listeners attached to the window object — listeners that OpenLayers registered internally and that Angular knows nothing about.

The div was Angular's, but the canvas, the listeners, and the render loop belonged to OpenLayers. Angular destroyed its own template node. It had no way to know about the imperative objects that OpenLayers attached on top.

> **Map DOM is not Angular template content**  
> The misconception: 'If the map's container div is in my Angular template, Angular will clean up everything inside it when the component is destroyed.' This seems plausible because Angular does remove template-managed DOM. But it breaks down because OpenLayers imperatively creates child elements (canvas, viewport divs) and registers global event listeners that reference the Map instance. Angular only removes the top-level div node; the JavaScript object graph that keeps the canvas and listeners alive is invisible to Angular's teardown. Without explicitly calling map.setTarget(null) and disposing resources, the Map instance remains rooted in memory by its own internal listeners.

## Two Worlds, One DOM

To understand why the naive approach fails, you need to see the boundary between two systems that operate by completely different rules.

Angular runs on a declarative, zone-patched change detection cycle. When a browser event fires — a click, a timer, an XHR response — Zone.js intercepts it and tells Angular: something changed, check the component tree. Angular walks the tree, re-evaluates templates, updates the DOM, and stops. The DOM elements Angular manages are created and destroyed through template bindings. When a component is destroyed, Angular removes the DOM nodes it created and runs ngOnDestroy. That is the extent of its responsibility.

OpenLayers is a different beast. When you call new Map({ target }), OpenLayers imperatively creates a viewport div, a canvas element, and dozens of internal event listeners on the document, the window, and the target element. It starts its own render loop driven by requestAnimationFrame. When the map's view changes — pan, zoom, resize — OpenLayers re-renders on its own schedule. It does not ask Angular for permission. It does not trigger Angular change detection. It just draws.

The two systems collide on the DOM. Angular owns the container div because it was declared in the template. OpenLayers owns the canvas inside that div because it created it imperatively. Neither system knows about the other's internal state.

Before reading the formal pattern, consider this: if OpenLayers' internal pointer-move event fires twenty times per second during a map drag, and Zone.js patches every browser event, what happens to Angular's change detection cycle each time the user drags the map?

Think about it. If you do nothing, every single pointermove event that OpenLayers listens to is intercepted by Zone.js, which tells Angular to run change detection across the entire component tree — twenty times per second — even though no Angular template binding has changed. The map is rendering on its own. Angular is doing useless work.

So the naive integration has two failure modes, not one. The first is a memory leak: the map is not torn down. The second is a performance leak: map events trigger Angular change detection unnecessarily. Both stem from the same root cause — OpenLayers is an external imperative system that lives outside Angular's change detection tree.

Here is the mental model that resolves both problems. The map is a long-lived mutable object. Rendering updates bypass Angular entirely — OpenLayers handles its own canvas redraws. But initialization and teardown must be explicitly synchronized with Angular's component lifecycle hooks, because the container div is Angular-managed. You are the bridge between two systems. Neither manages the other.

This is the realization that changes everything: the OpenLayers map is not Angular state. It is a foreign object that Angular happens to host. You must manage its birth in ngAfterViewInit — after the container exists — and its death in ngOnDestroy — before the container is removed. And you must prevent its events from waking up Angular's change detection by running them outside the zone.

This is the boundary the two systems share. Angular's template creates and destroys the container div. Everything else — the canvas, the render loop, the event listeners, the layers — is OpenLayers territory. You own the border crossing.

> **Map events do not need to trigger Angular change detection**  
> The misconception: 'Any browser event that fires while an Angular component is alive will trigger change detection, so OpenLayers map events will naturally cause Angular to update the component tree.' This is true by default — and that is exactly the problem. Without NgZone.runOutsideAngular, every pointermove, wheel, and touch event that OpenLayers binds is intercepted by Zone.js, which schedules a change detection cycle. For a map that fires dozens of events per second during interaction, this floods Angular with useless work. The fix is to register OpenLayers event listeners outside Angular's zone so they execute without notifying the change detector.

## The Correct Initialization Pattern

Now that the boundary is clear, the correct integration pattern follows from it logically. The map needs a container that exists in the DOM. Angular's ngOnInit fires before the view is rendered — the template has not been materialized yet, so the container div does not exist. ngAfterViewInit is the earliest lifecycle hook where ViewChild references are resolved and the template's DOM is available. That is where the map must be initialized.

Similarly, the map must be torn down before Angular removes the container. ngOnDestroy fires before the DOM is cleaned up, which gives you a window to call map.setTarget(null), severing the map's connection to the DOM element and allowing OpenLayers to remove its internal event listeners and stop its render loop.

Here is the minimal correct component. It uses ViewChild to get a typed reference to the container div, initializes the map in ngAfterViewInit, and calls setTarget(null) in ngOnDestroy.

``` TypeScript
import {
  Component,
  AfterViewInit,
  OnDestroy,
  ViewChild,
  ElementRef,
} from '@angular/core';
import Map from 'ol/Map';
import View from 'ol/View';
import TileLayer from 'ol/layer/Tile';
import OSM from 'ol/source/OSM';

@Component({
  selector: 'app-map-host',
  template: `<div #mapContainer class="map-container"></div>`,
  styles: [`
    .map-container {
      width: 100%;
      height: 100%;
    }
  `],
})
export class MapHostComponent implements AfterViewInit, OnDestroy {
  @ViewChild('mapContainer')
  private mapContainerRef!: ElementRef<HTMLDivElement>;

  private map!: Map;

  ngAfterViewInit(): void {
    this.map = new Map({
      target: this.mapContainerRef.nativeElement,
      layers: [
        new TileLayer({ source: new OSM() }),
      ],
      view: new View({
        center: [0, 0],
        zoom: 2,
      }),
    });
  }

  ngOnDestroy(): void {
    this.map.setTarget(null);
  }
}

```

Before moving on, compare this with the naive version. What changed? The id selector was replaced with ViewChild. The ngOnInit became ngAfterViewInit. And ngOnDestroy was added with setTarget(null). Each change maps directly to the mental model you built: the container must exist, and the map must be disconnected before the container is destroyed.

The tempting wrong model here is to use @ViewChild in ngOnInit — after all, the component class is initialized, so why wouldn't the reference be ready? Try it. Add a console.log(this.mapContainerRef) inside ngOnInit. It will be undefined. Angular has not yet rendered the template, so the element does not exist. ViewChild is resolved after the view is created, which is what ngAfterViewInit signals.

Another common mistake: calling setTarget(undefined) or simply letting the component be destroyed without calling anything. Without setTarget(null), the map retains its reference to the DOM element, its event listeners stay registered on the document and window, and the render loop continues. The map is functionally orphaned but still alive in memory.

> **ViewChild is not resolved in ngOnInit**  
> The misconception: 'ViewChild references are available as soon as the component class is instantiated, so you can use them in ngOnInit.' This seems plausible — the property is declared on the class, so it should be populated early. But Angular resolves ViewChild queries after the view DOM is created, which happens between ngOnInit and ngAfterViewInit. Accessing the reference in ngOnInit yields undefined. The correct hook for any DOM-dependent initialization is ngAfterViewInit.

> **setTarget(null) is not optional**  
> The misconception: 'If I do not call setTarget(null), Angular will clean up the DOM and the map will be garbage-collected.' Angular does remove the container div from the document. But the Map instance holds references to internal canvas elements, viewport divs, and registered event listeners on the window and document. Those references keep the Map alive in the JavaScript heap. Calling setTarget(null) explicitly severs the map's connection to the DOM and triggers OpenLayers' internal cleanup, removing listeners and stopping the render loop. Without it, the map leaks.

## The Complete Lifecycle in Practice

Now you will build the complete component, including NgZone management for OpenLayers events. This is the version you will carry forward into the rest of the project.

Consider what happens when you add a pointermove listener to the map — for example, to display coordinates as the mouse moves over the tunnel corridor. Without NgZone, each pointermove event triggers Angular change detection. During a drag, that is dozens of events per second. The component tree is checked and re-checked, even though the only thing that changed is a coordinate string that OpenLayers already rendered.

The fix is to register OpenLayers event listeners outside Angular's zone using NgZone.runOutsideAngular. If the listener does need to update Angular state — for instance, setting a component property that appears in the template — you selectively re-enter the zone with NgZone.run.

Before reading the full implementation, predict: for a pointermove listener that updates a currentCoordinates string shown in the template, which parts should run outside the zone and which should run inside?

The event listener registration itself should run outside the zone so that the high-frequency pointermove events do not trigger change detection. But the actual template update — setting currentCoordinates — should be wrapped in NgZone.run so that Angular knows to check the template. The expensive part (the event firing) stays outside. The meaningful part (a template binding changing) enters the zone.

``` TypeScript
// Complete MapHostComponent with NgZone management and full cleanup

import {
  Component,
  AfterViewInit,
  OnDestroy,
  ViewChild,
  ElementRef,
  NgZone,
} from '@angular/core';
import Map from 'ol/Map';
import View from 'ol/View';
import TileLayer from 'ol/layer/Tile';
import OSM from 'ol/source/OSM';
import { fromLonLat } from 'ol/proj';

@Component({
  selector: 'app-map-host',
  template: `
    <div #mapContainer class="map-container"></div>
    <div class=
```

``` TypeScript
// Complete MapHostComponent with NgZone management and full cleanup

import {
  Component,
  AfterViewInit,
  OnDestroy,
  ViewChild,
  ElementRef,
  NgZone,
} from '@angular/core';
import Map from 'ol/Map';
import View from 'ol/View';
import TileLayer from 'ol/layer/Tile';
import OSM from 'ol/source/OSM';

@Component({
  selector: 'app-map-host',
  template: `
    <div #mapContainer class="map-container"></div>
    <div class="coord-display">{{ currentCoordinates }}</div>
  `,
  styles: [`
    .map-container { width: 100%; height: 400px; }
    .coord-display { padding: 8px; font-family: monospace; }
  `],
})
export class MapHostComponent implements AfterViewInit, OnDestroy {
  @ViewChild('mapContainer')
  private mapContainerRef!: ElementRef<HTMLDivElement>;

  private map!: Map;
  private pointerMoveKey: any;
  currentCoordinates = '';

  constructor(private ngZone: NgZone) {}

  ngAfterViewInit(): void {
    this.map = new Map({
      target: this.mapContainerRef.nativeElement,
      layers: [
        new TileLayer({ source: new OSM() }),
      ],
      view: new View({
        center: [0, 0],
        zoom: 2,
      }),
    });

    // Register the listener outside Angular's zone
    this.ngZone.runOutsideAngular(() => {
      this.pointerMoveKey = this.map.on('pointermove', (evt) => {
        const coord = evt.coordinate;
        // Only re-enter the zone when template state actually changes
        this.ngZone.run(() => {
          this.currentCoordinates = `[${coord[0].toFixed(2)}, ${coord[1].toFixed(2)}]`;
        });
      });
    });
  }

  ngOnDestroy(): void {
    // Remove the specific event listener
    if (this.pointerMoveKey) {
      this.map.un(this.pointerMoveKey.type, this.pointerMoveKey.listener);
    }

    // Sever the map from the DOM and stop the render loop
    this.map.setTarget(null);

    // Dispose layers and their sources to release tile caches and feature data
    this.map.getLayers().forEach((layer) => {
      const source = layer.getSource();
      if (source) {
        source.dispose();
      }
      layer.dispose();
    });

    // Dispose the map itself
    this.map.dispose();
  }
}

```

Walk through the teardown sequence. First, the specific pointermove listener is removed with map.un. Then setTarget(null) severs the map from the DOM — OpenLayers removes its internal listeners on the target element and stops the render loop. Then each layer's source is disposed, releasing tile caches and any feature data held in memory. Finally, map.dispose() clears the map's internal state. This is a full, ordered teardown: listeners, DOM connection, data, then the map object itself.

A common mistake is to call setTarget(null) but skip disposing layers and sources. The map is disconnected from the DOM, but the tile layer still holds a cache of rendered tiles, and the source still holds feature data. Those caches can be significant — a tunnel control system with thousands of equipment features would leak megabytes of feature geometry data if sources are not disposed.

> **Listeners registered with map.on must be removed with map.un**  
> The misconception: 'If I registered the listener with map.on, calling setTarget(null) in ngOnDestroy will remove it automatically.' setTarget(null) does remove listeners that OpenLayers attaches to the target element internally. But listeners you registered yourself via map.on('pointermove', ...) are on the Map object, not the target element. They must be removed explicitly with map.un() or by calling dispose(). If you skip this, the listener function — and any closures it captures — remains in memory.

> **setTarget(null) alone does not dispose layers and sources**  
> The misconception: 'Calling setTarget(null) is sufficient cleanup. The layers and sources will be garbage-collected.' The Map instance may be disconnected from the DOM, but Layer and Source objects still hold references to tile caches, feature arrays, and internal state. Without calling source.dispose() and layer.dispose(), those caches persist. In a tunnel control system where sources may hold thousands of equipment features, this leaks substantial memory. The correct teardown sequence is: remove listeners, setTarget(null), dispose each source, dispose each layer, dispose the map.

Now try this yourself. Take the minimal component from the previous section and add a singleclick listener that logs the clicked coordinate to the console. Register it outside the zone. Do not re-enter the zone — just log. Navigate to the component, click the map, navigate away, and check DevTools for leaked listeners. Then add map.un for the click listener in ngOnDestroy and verify the leak is gone.

This exercise isolates the listener lifecycle from the template-update concern. If you can register, use, and remove a map event listener without leaking, you have the core skill needed for the tunnel control system's interaction layer.

## What You Now Own

The integration pattern you just built rests on a clear division of responsibility. Angular creates and destroys the container. OpenLayers creates and destroys everything inside it. You synchronize the two lifecycles and manage the boundary.

Here is the ownership map, distilled to its essentials.
- Angular owns: the template, the container element, and the component's own lifecycle hooks.
  - Angular owns the container div (declared in template, removed on component destruction).
- You own: the Map instance, its layers, its sources, its event listeners, and the zone boundary.
  - You call new Map() in ngAfterViewInit, not ngOnInit, because the container must exist.
  - You call setTarget(null) in ngOnDestroy, not rely on Angular, because the map is not template content.
  - You wrap event listeners in NgZone.runOutsideAngular because map events do not need to trigger change detection.
  - You dispose layers, sources, and the map itself because caches and feature data are not garbage-collected otherwise.

Carry this checklist into every component that hosts an OpenLayers map.
1. ViewChild reference to the container div — not an id string lookup.
2. new Map() called in ngAfterViewInit — not ngOnInit, not the constructor.
3. Event listeners registered via NgZone.runOutsideAngular — with selective NgZone.run for template updates only.
4. ngOnDestroy calls map.un for each registered listener.
5. ngOnDestroy calls map.setTarget(null) to sever DOM connection and stop the render loop.
6. ngOnDestroy disposes each layer's source, then each layer, then the map itself.

You now have a MapHostComponent that initializes safely, runs efficiently, and tears down completely. The map is empty — a base tile layer with no tunnel data. In the next lesson, you will add a vector layer containing the tunnel line, portal points, and a small set of equipment features. That means creating sources, features, and geometries — and the ownership rules you learned here will govern how those objects are created, updated, and disposed within Angular's lifecycle.

## Conclusion
You began by witnessing the failure of naive integration: a map created in ngOnInit leaks canvas elements and event listeners because Angular never tears it down. That failure revealed that OpenLayers is an external imperative system — its Map renders on its own animation frame, bypassing Angular's change detection entirely. The correct integration pattern uses ngAfterViewInit to initialize after the container exists, NgZone.runOutsideAngular to prevent map events from flooding Angular's change detection, and map.setTarget(null) in ngOnDestroy to sever the map from the DOM and release listeners. The map is long-lived state that Angular does not manage — you must own its lifecycle explicitly.

**Next:**  
*In the next lesson, you will add a base tile layer and a vector layer containing a tunnel line, portals, and a small set of point equipment features — building on the lifecycle scaffolding you now have.*
