<!--
  Copyright 2015-2026 Ritense BV, the Netherlands.
  Licensed under EUPL, Version 1.2 (the "License");
  https://joinup.ec.europa.eu/collection/eupl/eupl-text-eupl-12
-->

# Using the side panel

> **Audience:** frontend developers extending a Valtimo Angular application. External plugins use the SDK instead; see [Side panels](../external-plugins/develop-a-plugin.md#side-panels). For the end-user view, see [What is the side panel?](https://docs.valtimo.nl/fundamentals/side-panel).

Valtimo has one side panel at the right edge of the internal layout. Any Angular code can show a component in it through `SidePanelService` from `@valtimo/components`. The case detail task list and external plugin panels both use this API.

## Overview

`<valtimo-side-panel>` is rendered once in `layout-internal`, next to the router outlet. It does not know what it shows: providers **offer** content, and the latest offer takes over the panel.

| Concept | Description |
|---------|-------------|
| Offer | A `SidePanelOffer`: a component plus title, inputs and options. `offer()` returns a `SidePanelHandle`. |
| Key | Identifies the content. Re-offering the current key updates title and inputs and keeps the component alive. A different key destroys the current component and replaces it. |
| Dismiss | The user's close button. Hides the panel; the component stays alive. |
| Withdraw | Removes the content and destroys the component. Only the provider does this. |

{% hint style="warning" %}
The panel never withdraws content on its own, not even on navigation. Withdraw it when it stops applying, typically in `ngOnDestroy` of the component that offered it.
{% endhint %}

Because the panel lives outside the routed page, its content survives navigation. Showing and hiding is CSS-only, so an iframe in the panel does not reload. The panel does not survive a browser reload.

## Offering content

```typescript
import {Component, Injector, Input, OnDestroy} from '@angular/core';
import {SidePanelHandle, SidePanelService} from '@valtimo/components';
import {ButtonModule} from 'carbon-components-angular';

@Component({
  standalone: true,
  selector: 'app-case-notes',
  template: `<p>Notes for case {{ documentId }}</p>`,
})
export class CaseNotesComponent {
  @Input({required: true}) documentId!: string;
}

@Component({
  standalone: true,
  selector: 'app-case-notes-button',
  imports: [ButtonModule],
  template: `<button cdsButton="ghost" (click)="openNotes()">Notes</button>`,
})
export class CaseNotesButtonComponent implements OnDestroy {
  @Input({required: true}) caseDefinitionKey!: string;
  @Input({required: true}) documentId!: string;

  private handle: SidePanelHandle | null = null;

  constructor(
    private readonly injector: Injector,
    private readonly sidePanelService: SidePanelService
  ) {}

  public openNotes(): void {
    this.handle = this.sidePanelService.offer({
      key: `case-notes-${this.documentId}`,
      title: 'Notes',
      component: CaseNotesComponent,
      inputs: {documentId: this.documentId},
      injector: this.injector,
      origin: {
        label: 'this case',
        route: ['/cases', this.caseDefinitionKey, 'document', this.documentId],
      },
    });
    this.handle.replaced$.subscribe(() => (this.handle = null));
  }

  public ngOnDestroy(): void {
    this.handle?.withdraw();
  }
}
```

- Pass `injector` when the content needs services provided by the page that offers it, such as route-scoped or component-provided services. Without it, the content only resolves root providers.
- Calling `openNotes()` again with the same key returns the same handle and shows the panel again if the user had closed it. Offering on every load is safe.
- `handle.withdraw()` does nothing once the content has been replaced, so it never removes another provider's content. `SidePanelService.withdraw(key)` behaves the same way and removes the content only while it still has that key.

## `SidePanelOffer`

| Field | Type | Description |
|-------|------|-------------|
| `key` | `string` | Content identity; see [Overview](#overview). |
| `title` | `string` | Panel header title. |
| `subtitle` | `string` | Optional line below the title. |
| `subtitleLink` | `string \| string[]` | Turns the subtitle into a `routerLink`. |
| `component` | `Type<unknown>` | Component rendered in the panel body. |
| `inputs` | `Record<string, unknown>` | Inputs for `component`. |
| `injector` | `Injector` | Parent injector for `component`. |
| `origin` | `{label: string \| Observable<string>; route: string[]}` | Shown as "For *label*", linking to `route`. Marked "Not this page" while the user is on another route. The label can be an observable that resolves after the offer. |
| `pinned` | `boolean` | Shows a pin; see [Pinning](#pinning). |
| `minWidth` | `number` | Widens the panel while this content shows, without changing the user's preferred width. |
| `width` | `number` | The content's own width, used instead of the shared preferred width. You persist it yourself; see [Width](#width). |

## `SidePanelHandle`

| Member | Description |
|--------|-------------|
| `dismissed$` | Emits when the user closes the panel. |
| `replaced$` | Emits once when another key takes over. All handle streams then complete. |
| `resized$` | Emits the final width after the user resizes the panel while showing this content. |
| `hide()` | Hides the panel and keeps the content alive. Re-offer the key to show it again. |
| `withdraw()` | Removes the content, if it is still current. |

`SidePanelService` also exposes `current$`, `visible$` and `width$`, plus `dismiss()` and `reopen()`.

## Pinning

Set `pinned: true` to mark the content with a pin. Its tooltip tells the user the panel stays open when they go to other pages.

The pin is only a marker. The panel does not act on it, so the promise is yours to keep. Pin content that you keep offered across pages, such as an external plugin panel. Don't pin content that you withdraw when its page is left, such as the case task panel.

## Width

The user resizes the panel by dragging its left edge or with the arrow keys. Width is clamped to 320–960 px and at most 60% of the viewport.

- **Default:** a shared preferred width, stored in `localStorage` under `sidePanelWidth` (default 480 px).
- **Own width:** set `width` on the offer. The shared preference is left untouched. Save the value from `resized$` and pass it back in the next offer. The case task panel does this to store its width in the user settings.

```typescript
handle.resized$.subscribe(width => this.settingsService.saveNotesPanelWidth(width));
```

## Layout integration

On large screens the panel pushes the page aside instead of covering it. It publishes its occupied width as the CSS custom property `--valtimo-side-panel-offset` on `<html>`, and sets it to `0px` while hidden and on small screens, where the panel overlays the page. Custom fixed-position elements that should stay clear of the panel can use it:

```scss
.my-floating-toolbar {
  position: fixed;
  right: calc(2rem + var(--valtimo-side-panel-offset, 0px));
}
```

## Reference implementation

`CaseDetailTaskPanelService` (`frontend/projects/valtimo/case/src/lib/services/case-detail-task-panel.service.ts`) shows the full pattern:

- It is provided in the case detail component and offers with that component's injector.
- It hides the panel on tabs without tasks and re-offers when tasks apply again.
- It withdraws in `ngOnDestroy`.
- It stores its own width from `resized$`.
