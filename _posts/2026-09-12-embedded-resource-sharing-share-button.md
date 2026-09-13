---
layout: post
title: "Share without the detour: embedding the resource sharing button in OpenSearch Dashboards"
authors:
  - dchanp
date: 2026-09-12
categories:
  - technical-post
meta_keywords: security, resource sharing, access control, dashboards, share button, extensibility, plugins, authorization
meta_description: "Learn how, in OpenSearch 3.9, OpenSearch Dashboards plugins can surface the centralized resource sharing Share button directly in their own pages—so you can share a resource without leaving the page it lives on."
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

Sharing a resource in OpenSearch Dashboards used to mean leaving the page you were on: open the central Resource Access Management app, select the resource type, and find your resource in a table. In OpenSearch 3.9, you can skip that detour and share a resource directly from the page where it already lives.

The [resource sharing framework]({{ site.baseurl }}/blog/Introducing-Resource-Sharing/) introduced in an earlier post answered *who can access what* for plugin-defined resources such as anomaly detectors, ML models, and report definitions. This post is about *where* you do the sharing: an embedded **Share button** that shows up wherever a resource already lives—a table row, a page header, a details flyout.

---

## See it in action

The following image shows the Share button in the **Access** column of a resource list. Each row also shows whether the resource is private or shared, so you can read the sharing state at a glance:

![The resource sharing Share button in the Access column of a resource list](/assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/access-column.png)

Selecting the button opens the same access modal that the central app uses, so the flow is identical no matter which plugin you're in:

![The resource sharing access modal for managing who a resource is shared with](/assets/media/blog-images/2026-09-12-embedded-resource-sharing-share-button/access-modal.png)

You can also [watch a short screen recording of the Share button in action](https://github.com/user-attachments/assets/d659a14c-864e-4fc4-9fb5-9ec1de2bf4a8). The button is data-source aware, too: in a multiple data sources (MDS) deployment, it targets the cluster you're working in, as shown in [this demo](https://github.com/opensearch-project/security-dashboards-plugin/pull/2520#issuecomment-5611229423).

---

## The same experience in every plugin

Every plugin that owns shareable resources shows the same Share button and opens the same access modal, so sharing works the same way whether you're in the Anomaly Detection plugin, the Reporting plugin, or the Security Analytics plugin. There's one place to learn, and it behaves consistently everywhere.

Just as important is what happens when security *isn't* in the picture. Security is optional in OpenSearch—a cluster can run without it, or with resource sharing turned off—so the button was designed to require no dependency on the Security plugin at all. It appears when resource sharing is available and stays out of the way when it isn't, with nothing for a plugin to configure or turn on.

---

## How a plugin adds it

Adoption is deliberately small. A plugin marks the spot where it wants the button by rendering a placeholder element:

```jsx
<div
  data-resource-share-button
  data-resource-id={detector.id}
  data-resource-type="anomaly-detector"
/>
```

That's the whole integration—no imports, no plugin dependency, and no manifest changes. When the Security plugin is present and resource sharing is enabled, it finds these placeholders and fills in the button, wiring up the sharing state and permission checks. The idea mirrors how the resource sharing backend already works: *declare yourself, and the framework fulfills it*. When security is absent, the feature is turned off, or you don't have permission to share, the button simply doesn't appear (or appears disabled, with a short explanation).

Plugin authors can find the full set of options—a compact icon for table rows, multiple data sources support, and more—in [security-dashboards-plugin#2491](https://github.com/opensearch-project/security-dashboards-plugin/pull/2491).

---

## Plugins that support the Share button today

Not every plugin manages shareable resources, but the ones that do are already onboarded. The embedded Share button is available in the following plugins:

* The **Alerting plugin** (monitors and composite monitors)
* The **Anomaly Detection plugin** (detectors and forecasters)
* The **Flow Framework plugin** (workflows)
* The **ML Commons plugin** (model groups)
* The **Notifications plugin** (channels)
* The **Reporting plugin** (report definitions and reports)
* The **Security Analytics plugin** (detectors and correlation rules)

Because there is no dependency to adopt, any future plugin that introduces a shareable resource type can join this list with the same one-line change.

---

## Next steps

If your cluster already uses resource sharing, the Share button shows up inline as these plugins adopt it—no extra configuration beyond enabling resource sharing. To go deeper, see the following resources:

* [Introducing resource sharing: A new access control model for OpenSearch]({{ site.baseurl }}/blog/Introducing-Resource-Sharing/)
* [Resource sharing and access control documentation](https://docs.opensearch.org/)

If your plugin manages shareable resources, adding the Share button is one of the smallest integrations in OpenSearch and a good first contribution. We welcome your feedback on the [OpenSearch forum](https://forum.opensearch.org/).
