---
layout: post
title: "Share without the detour: embedding the resource sharing button in OpenSearch Dashboards"
authors:
  - dchanp
date: 2026-09-12
categories:
  - technical-post
meta_keywords: security, resource sharing, access control, dashboards, share button, extensibility, plugins, authorization
meta_description: "Learn how, in OpenSearch 3.9, OpenSearch Dashboards plugins can surface the centralized resource sharing Share button directly in their own pages with a single DOM marker—no plugin dependency, no imports, and no manifest changes."
tags:
  - security
  - access control
  - resource sharing
  - dashboards
  - anomaly detection
  - ml commons
  - security analytics
  - opensearch 3.9
---

Sharing a resource in OpenSearch Dashboards used to mean leaving whatever you were doing: open the central Resource Access Management app, pick the resource type, and find your resource in a table. In OpenSearch 3.9, you can skip that detour and share a resource right from the page it already lives on.

The [resource sharing framework]({{ site.baseurl }}/blog/Introducing-Resource-Sharing/) introduced in an earlier post answered *who can access what* for plugin-defined resources such as anomaly detectors, ML models, and report definitions. This post is about *where* you do the sharing: an embedded **Share button** that shows up wherever a resource already lives—a table row, a page header, a details flyout—and that a plugin can adopt with a single line of markup and no dependency on the Security plugin.

---

## See it in action

The following image shows the Share button in the **Access** column of a resource list. Each row also shows whether the resource is private or shared, so you can read the sharing state at a glance:

![The resource sharing Share button in the Access column of a resource list](/assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/access-column.png)

Selecting the button opens the same access modal that the central app uses, so the flow is identical no matter which plugin you're in:

![The resource sharing access modal for managing who a resource is shared with](/assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/access-modal.png)

You can also [watch a short screen recording of the Share button in action](https://github.com/user-attachments/assets/d659a14c-864e-4fc4-9fb5-9ec1de2bf4a8), from [security-dashboards-plugin#2491](https://github.com/opensearch-project/security-dashboards-plugin/pull/2491). The button is data-source aware, too: pass a data source ID and it targets the correct cluster in a multiple data sources (MDS) deployment, as shown in [security-dashboards-plugin#2520](https://github.com/opensearch-project/security-dashboards-plugin/pull/2520) and its [demo video](https://github.com/opensearch-project/security-dashboards-plugin/pull/2520#issuecomment-5611229423).

---

## One experience, without coupling every plugin to security

The backend already had a clean extension point: a plugin registers its resource types with the Security plugin, and the framework takes care of authorization, sharing records, and auditing. The UI was the missing half.

The goal was easy to state and annoying to satisfy: every plugin that owns shareable resources should offer the *same* sharing experience—same button, same modal, same private-or-shared status, same permission checks—without taking a dependency on the Security plugin. Security is optional in OpenSearch. A cluster might run without it, or with resource sharing turned off, and every plugin has to keep working either way.

Two obvious designs didn't hold up:

* A **shared React component** would make every consumer import from, and depend on, the Security plugin—and break builds on clusters where security isn't installed.
* A **registry or service** that plugins call at startup still means a dependency and some lifecycle choreography.

What held up was a design with no coupling at all: the consumer decides where the button goes, and the button quietly doesn't render when security or resource sharing isn't there.

---

## How the DOM-marker SPI works

The embedded Share button is delivered through a **DOM-marker service provider interface (SPI)**. It borrows the same idea the backend uses for resource types—*declare yourself, and the framework fulfills it*:

> A plugin renders an empty marker element wherever it wants a Share button. The Security plugin finds those markers and mounts the button into them.

A consumer renders an element with a few `data-` attributes (this example targets OpenSearch 3.9):

```jsx
// In any OpenSearch Dashboards plugin — no imports or dependencies needed.
<div
  data-resource-share-button
  data-resource-id={detector.id}
  data-resource-type="anomaly-detector"
/>
```

That's the whole contract, and it behaves the same in a table cell, a page header, or a flyout.

When the Security plugin is present and resource sharing is enabled, it does the following:

1. Scans the page for elements carrying the `data-resource-share-button` marker.
2. Mounts the Share button into each one and wires up the sharing record, access levels, and permission checks.
3. Watches for markers that appear, change, or disappear (using a `MutationObserver`) as you page through tables and move around the application.

The consumer never imports the Security plugin's APIs, and the Security plugin never learns anything about the consumer: the DOM is the entire interface. Rendering one button per row stays cheap, too—the buttons on a page coalesce their lookups into a single request instead of each fetching on its own.

### The marker contract

The following attributes control the button:

| Attribute | Required | Purpose |
| --- | --- | --- |
| `data-resource-share-button` | Yes | Marks the element as a share-button placeholder. |
| `data-resource-id` | Yes | The ID of the resource to share. |
| `data-resource-type` | Yes | The registered resource type (for example, `anomaly-detector`). |
| `data-resource-data-source-id` | No | The data source ID, for multiple data sources (MDS) deployments. |
| `data-resource-name` | No | A human-readable name to show in the modal instead of the ID. |
| `data-resource-share-display` | No | `button` (default, labeled) or `icon` (compact, for dense table rows). |
| `data-resource-hide-status` | No | Hides the private or shared status pill (for example, on a details page where the status is shown elsewhere). |

---

## What happens when security is disabled

Because the Security plugin fills in a marker only when the marker applies, the feature degrades gracefully instead of erroring:

* **Security is absent, or resource sharing is disabled**: the marker stays inert, so the consumer renders nothing extra and behaves exactly as before.
* **The resource type isn't registered for sharing**: the mounted button hides itself.
* **You don't have permission to share**: the button appears disabled, with a tooltip that explains why.

There's nothing to feature-flag and nothing to conditionally import—the same code path runs with or without security.

---

## Adding the button to a resource list

The most common place to surface sharing is a resource list, which usually comes down to one conditional column that renders the marker in `icon` mode:

```jsx
{
  field: 'id',
  name: 'Access',
  render: (id, item) => (
    <div
      data-resource-share-button
      data-resource-id={id}
      data-resource-type="anomaly-detector"
      data-resource-name={item.name}
      data-resource-share-display="icon"
    />
  ),
}
```

Two things worth knowing from wiring this up across plugins:

* For dense tables, use `icon` display and let the table size the column to its content. In EUI's `EuiInMemoryTable` and `EuiBasicTable`, `tableLayout="auto"` keeps a narrow, fixed-width column from clipping the button.
* Add the column only when the resource type is registered for sharing, so the UI stays clean on clusters where the feature is off.

---

## Plugins that support the Share button today

Not every plugin has shareable resources, but the ones that do are already onboarded. The embedded Share button is live in the following plugins:

* The **Reporting plugin** (report definitions and reports)
* The **Notifications plugin** (channels)
* The **Security Analytics plugin** (detectors and correlation rules)
* The **Anomaly Detection plugin** (detectors and forecasters)
* The **ML Commons plugin** (model groups)

Because there's no dependency to take on, any future plugin with a shareable resource type can join this list with the same one-line change.

---

## Next steps

If your cluster already uses resource sharing, the Share button shows up inline as these plugins adopt the marker—no extra configuration beyond enabling resource sharing. To go deeper, see the following resources:

* [Introducing resource sharing: A new access control model for OpenSearch]({{ site.baseurl }}/blog/Introducing-Resource-Sharing/)
* [Resource sharing and access control documentation](https://docs.opensearch.org/)

If your plugin owns shareable resources, wiring up the Share button is about as small as an OpenSearch integration gets—and it makes a great first contribution. We'd welcome your feedback on the [OpenSearch forum](https://forum.opensearch.org/).
