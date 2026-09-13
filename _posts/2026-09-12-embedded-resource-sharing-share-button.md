---
layout: post
title: "Share from any page: the embedded resource sharing button in OpenSearch Dashboards"
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

In this blog post, you'll learn how OpenSearch 3.9 lets you share a resource from the page you're already on—no detour to a separate screen required.

In [Introducing resource sharing: A new access control model for OpenSearch]({{ site.baseurl }}/blog/Introducing-Resource-Sharing/), we introduced owner-controlled, fine-grained sharing for plugin-defined resources such as anomaly detectors, ML models, and report definitions. That framework answered the question of *who can access what*. This post is about *where* you do the sharing.

When you're looking at a list of detectors, the most natural moment to share one is right there, next to it—not after navigating to a separate screen. The embedded **resource sharing Share button** puts the sharing experience wherever a resource already lives in OpenSearch Dashboards: a table row, a page header, or a details flyout. And for the plugins that own those resources, adding it takes a single line—with no dependency on the Security plugin, no imports, and no manifest changes.

---

## See it in action

The following image shows the Share button in the **Access** column of a resource list, where each row also shows whether the resource is private or shared:

![The resource sharing Share button in the Access column of a resource list](/assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/access-column.png)

Selecting the button opens the same access modal used throughout OpenSearch Dashboards, so the experience is identical no matter which plugin you're in:

![The resource sharing access modal for managing who a resource is shared with](/assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/access-modal.png)

You can also [watch a short screen recording of the Share button in action](https://github.com/user-attachments/assets/d659a14c-864e-4fc4-9fb5-9ec1de2bf4a8), from [security-dashboards-plugin#2491](https://github.com/opensearch-project/security-dashboards-plugin/pull/2491). The button is also aware of multiple data sources (MDS): pass a data source ID and it targets the correct cluster in multi-cluster deployments, as shown in [security-dashboards-plugin#2520](https://github.com/opensearch-project/security-dashboards-plugin/pull/2520) and its [demo video](https://github.com/opensearch-project/security-dashboards-plugin/pull/2520#issuecomment-5611229423).

---

## One experience, without coupling every plugin to security

The resource sharing backend already had a clean extension point: plugins register their resource types with the Security plugin, and the framework handles authorization, sharing records, and auditing. The open question was the UI.

We wanted every plugin that manages shareable resources to offer the same sharing experience—the same button, the same modal, the same private or shared status, and the same permission checks. But there was a hard constraint: a consumer plugin shouldn't have to take a build-time or runtime dependency on the Security plugin. Security is optional in OpenSearch—a cluster may run without it, or with resource sharing disabled—and every plugin has to keep working in all of those configurations.

Two obvious approaches fell short:

* A **shared React component** would force every consumer to import from, and depend on, the Security plugin—breaking builds on clusters where security isn't present.
* A **registry or service** that plugins call at startup still requires a dependency and careful lifecycle coordination.

What we wanted instead was no coupling at all: the consumer owns placement, and the button simply isn't rendered when security or resource sharing is unavailable.

---

## How the DOM-marker SPI works

The embedded Share button uses a **DOM-marker service provider interface (SPI)**. On the UI side, it mirrors the "implement the interface" philosophy that the backend already uses for resource types:

> A plugin declares *where* it wants a Share button by rendering an empty marker element. The Security plugin discovers those markers and fills them in.

A consumer renders an element with a few `data-` attributes (this example uses OpenSearch 3.9):

```jsx
// In any OpenSearch Dashboards plugin — no imports or dependencies needed.
<div
  data-resource-share-button
  data-resource-id={detector.id}
  data-resource-type="anomaly-detector"
/>
```

That's the whole contract, and it works just as well in a table cell, a page header, or a flyout.

When the Security plugin is present and resource sharing is enabled, it does the following:

1. Scans the page for elements that carry the `data-resource-share-button` marker.
2. Mounts the centralized Share button into each one, wiring up the sharing record, access levels, and permission checks.
3. Watches for markers that appear, change, or disappear (using a `MutationObserver`) as you page through tables and move around the application.

Because discovery happens entirely through the DOM, the consumer needs no knowledge of the Security plugin's APIs, and the Security plugin needs no knowledge of the consumer.

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

Because the Security plugin fills in the button only when it applies, the feature degrades gracefully:

* **Security is absent, or resource sharing is disabled**: the marker stays empty, so the consumer renders nothing extra and behaves exactly as before.
* **The resource type isn't registered for sharing**: the mounted button hides itself.
* **You don't have share permission**: the button appears in a disabled state that explains why.

There's nothing for the consumer to feature-flag or conditionally import—the same code path works with and without security.

---

## Adding the button to a resource list

The most common place to surface sharing is a resource list, and that's usually a single conditional column that renders the marker in `icon` mode:

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

Two tips from adopting this across plugins:

* For dense list tables, use `icon` display and let the table size the column to its content. In EUI's `EuiInMemoryTable` and `EuiBasicTable`, setting `tableLayout="auto"` keeps a compact, fixed-width column from clipping the button.
* Add the column only when the resource type is registered for sharing, so the UI stays clean on clusters where the feature is off.

---

## Plugins that support the Share button today

Not every plugin manages shareable resources, but the ones that do are already onboarded. The embedded Share button appears in these plugins, each with a single-line marker:

* The **Reporting plugin** (report definitions and reports)
* The **Notifications plugin** (channels)
* The **Security Analytics plugin** (detectors and correlation rules)
* The **Anomaly Detection plugin** (detectors and forecasters)
* The **ML Commons plugin** (model groups)

Because the pattern is dependency-free, any future plugin that introduces a shareable resource type can adopt the button with the same one-line change.

---

## Next steps

If your cluster already uses resource sharing, the Share button appears inline as these plugins adopt the marker—no configuration is required beyond enabling resource sharing. To learn more, see the following resources:

* [Introducing resource sharing: A new access control model for OpenSearch]({{ site.baseurl }}/blog/Introducing-Resource-Sharing/)
* [Resource sharing and access control documentation](https://docs.opensearch.org/)

If your plugin manages shareable resources, adding the Share button is one of the easiest ways to contribute. We'd welcome your feedback on the [OpenSearch forum](https://forum.opensearch.org/).
