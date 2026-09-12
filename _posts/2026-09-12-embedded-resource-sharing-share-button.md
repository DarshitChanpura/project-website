---
layout: post
title: "Share from any page: bringing the resource sharing button into OpenSearch Dashboards"
authors:
  - cwperks
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

<!--
DRAFT — remaining items to confirm before publishing:
- date: set to the intended publish date (filename must match).
- authors list.
- add screenshot at assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/share-button.png
  (referenced in "See it in action"); optionally share-modal.png / ram-page.png.
-->

In [Introducing resource sharing: A new access control model for OpenSearch]({{ site.baseurl }}/blog/Introducing-Resource-Sharing/), we described how the Security plugin brings owner-controlled, fine-grained sharing to plugin-defined resources such as anomaly detectors, ML models, and report definitions. That work also introduced a centralized **Resource Access Management (RAM)** page—a single place to review and manage everything shared with you or by you.

A central page is the right home for managing sharing at scale, but it isn't always where you *think* about sharing. When you are looking at a list of detectors, the most natural moment to share one is right there, next to it—not after navigating away to a separate page. So the next step was to bring the sharing experience to wherever a resource already lives in OpenSearch Dashboards.

This post introduces the **embedded resource sharing Share button**, available in OpenSearch 3.9: a single, centralized Share control that any OpenSearch Dashboards plugin can drop into its own pages—a table row, a page header, a details flyout—with **no dependency on the Security plugin, no imports, and no manifest changes**.

---

## See it in action

![The resource sharing Share button embedded inline in a resource list](/assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/share-button.png)

Here the Share button sits right in the resource list; selecting it opens the same access modal used everywhere else in OpenSearch Dashboards. You can also [watch a short screen recording of the Share button in action](https://github.com/user-attachments/assets/d659a14c-864e-4fc4-9fb5-9ec1de2bf4a8), from [security-dashboards-plugin#2491](https://github.com/opensearch-project/security-dashboards-plugin/pull/2491).

The button is also multiple data sources (MDS)–aware: pass the data source id and it targets the correct cluster in multi-cluster deployments. See [security-dashboards-plugin#2520](https://github.com/opensearch-project/security-dashboards-plugin/pull/2520) and its [demo video](https://github.com/opensearch-project/security-dashboards-plugin/pull/2520#issuecomment-5611229423).

---

## The challenge: one experience, many plugins

The backend already had a clean extension point. Plugins register their resource types with the Security plugin, and the framework handles authorization, sharing records, and auditing. But the *UI* was an open question.

We wanted every plugin that manages shareable resources to offer the exact same Share affordance: the same button, the same access modal, the same private/shared status, and the same permission checks. We also had a hard constraint: consumer plugins should **not** need to take a build- or runtime dependency on the Security plugin. In OpenSearch, security is optional—a cluster may run without it, or with resource sharing disabled—and plugins must continue to work in all of those configurations.

The obvious approaches each had drawbacks:

* **A shared React component** would force every consumer to import from—and depend on—the Security plugin, breaking builds where security isn't present.
* **A registry/service** that plugins call into at startup still requires a dependency and careful lifecycle coordination.

We wanted something with zero coupling, where placement is entirely owned by the consumer and the feature simply *disappears* when security or resource sharing isn't available.

---

## The solution: a DOM-marker SPI

The embedded Share button uses a **DOM-marker service provider interface (SPI)**. It mirrors, on the UI side, the same "implement the interface" philosophy the backend uses for resource types:

> A plugin declares *where* it wants a Share button by rendering an empty marker element. The Security plugin discovers those markers and fills them in.

Concretely, a consumer renders an element carrying a few `data-` attributes:

```jsx
// In any OpenSearch Dashboards plugin — no imports, no dependency needed.
<div
  data-resource-share-button
  data-resource-id={detector.id}
  data-resource-type="anomaly-detector"
/>
```

That's the entire contract. The consumer owns placement completely—this snippet works just as well in a table cell, a page header, or a flyout.

On the other side, when the Security plugin is installed and resource sharing is enabled, it runs a small discovery loop that:

1. Scans the DOM for elements with the `data-resource-share-button` marker.
2. Mounts the centralized Share button into each one, wiring up the sharing record, access levels, and permission checks.
3. Uses a `MutationObserver` to track markers that appear, disappear, or change as users paginate tables and navigate the single-page app.

Because discovery is driven entirely by the DOM, the consumer needs no knowledge of the Security plugin's APIs, and the Security plugin needs no knowledge of the consumer.

### The marker contract

| Attribute | Required | Purpose |
| --- | --- | --- |
| `data-resource-share-button` | Yes | Marks the element as a share-button placeholder. |
| `data-resource-id` | Yes | The id of the resource to share. |
| `data-resource-type` | Yes | The registered resource type (for example, `anomaly-detector`). |
| `data-resource-data-source-id` | No | The data source id, for multiple data sources (MDS) deployments. |
| `data-resource-name` | No | A human-readable name to display in the modal instead of the id. |
| `data-resource-share-display` | No | `button` (default, labeled) or `icon` (compact, for dense table rows). |
| `data-resource-hide-status` | No | Hides the Private/Shared status pill (for example, on a details page where status is shown elsewhere). |

---

## Graceful by design

The marker approach makes the "security is optional" story clean:

* **Security absent or resource sharing disabled** → the marker stays empty. Consumers render nothing extra and behave exactly as before.
* **Resource type not registered/protected** → the mounted button hides itself.
* **User lacks share permission** → the button renders in a disabled state that explains why.

There is nothing for the consumer to feature-flag or conditionally import. The same code path works with and without security.

---

## Adopting it in a list table

A common place to surface sharing is a resource list. Adopting the button is typically a single conditional column that renders the marker in `icon` mode:

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

A couple of practical tips we learned while rolling this out across plugins:

* For dense list tables, prefer `icon` display and let the table size the column to its content (in EUI's `EuiInMemoryTable`/`EuiBasicTable`, `tableLayout="auto"` prevents a compact fixed-width column from clipping the button).
* Only show the column when the type is actually registered for sharing, so the UI stays clean on clusters where the feature is off.

---

## Where it's available

Not every plugin manages shareable resources—but the ones that do are already onboarded. The embedded Share button now appears across these OpenSearch Dashboards plugins, each with just a one-line marker:

* Reporting (report definitions and reports)
* Notifications (channels)
* Security Analytics (detectors and correlation rules)
* Anomaly Detection (detectors and forecasters)
* ML Commons (model groups)

Because the pattern is dependency-free, any future plugin that introduces a shareable resource type can adopt the button with the same one-line change—render the marker where it belongs, and the centralized experience takes over.

---

## Try it out

If you already use resource sharing, the Share button will start appearing inline as consumer plugins adopt the marker—no configuration required beyond enabling resource sharing on the cluster. To manage everything in one place, the centralized Resource Access Management page remains available in OpenSearch Dashboards.

![The central Resource Access Management page, showing a shared anomaly detector and its access level](/assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/ram-page.png)

To learn more:

* [Introducing resource sharing: A new access control model for OpenSearch]({{ site.baseurl }}/blog/Introducing-Resource-Sharing/)
* Resource sharing and access control documentation on [docs.opensearch.org](https://docs.opensearch.org/)

We would love your feedback on the [OpenSearch forum](https://forum.opensearch.org/) and welcome contributions—if your plugin introduces a shareable resource type, adding the Share button is one of the easiest ways to get involved.
