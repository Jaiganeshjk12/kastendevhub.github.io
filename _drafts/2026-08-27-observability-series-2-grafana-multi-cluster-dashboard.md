---
author: jaikarthikeyan
date: 2026-08-27 13:06:02 +0300
description: "In this blog, we will customize the public Kasten Grafana dashboard to provide multi-cluster visibility from a centralized Prometheus backend."
featured: false
image: "/images/posts/2026-08-27-observability-grafana-multi-cluster-dashboard/kasten-grafana-multi-cluster-dashboard.png"
image_caption: ""
layout: post
published: true
tags: [Kasten, Grafana, Metrics, Observability, Prometheus]
title: "Customizing the Kasten Grafana dashboard for multi-cluster visibility"
---

In [Part 1]({% post_url 2026-05-06-observability-series-1-prometheus-remote-writes %}), we configured Kasten's in-cluster Prometheus to send metrics to a centralized backend with `remote_write`. That solves getting metrics out of each cluster, but it does not provide a view of them.

Kasten already publishes a [Grafana dashboard](https://grafana.com/grafana/dashboards/21065-k10-dashboard/) for action completions and failures, action duration, resource utilization, and compliance. The dashboard is designed for a single Veeam Kasten instance. When it is pointed at the multi-cluster backend from Part 1, panels either blend every cluster into one result or double-count data in aggregations such as `sum()` and `increase()`.

This post shows how to retrofit the published dashboard with two template variables and cluster filtering, rather than build a replacement dashboard from scratch.

This guide is the second piece of a three-part series focused on building an end-to-end, backend-agnostic monitoring setup for Kasten:

- **Part 1**: [Prometheus Remote Write Configuration with Kasten]({% post_url 2026-05-06-observability-series-1-prometheus-remote-writes %}).
- **Part 2 (this post)**: Customizing the Public Kasten Grafana Dashboard for Multi-Cluster Visibility.
- **Part 3**: Setting up alerting based on Kasten's exported metrics.

By the end of the series, you will have a repeatable pattern for exporting Kasten metrics from multiple clusters, visualizing them in Grafana, and wiring up alerts.

## Why the published dashboard does not work with a central backend

Two assumptions in the published dashboard break when it is connected to aggregated, multi-cluster data.

#### Hardcoded data source

Every panel is bound to a data source named literally `Prometheus`. If your Grafana instance does not have a data source with that exact name, panels are broken when you import the dashboard. Even when it does, you cannot switch backends without editing every panel.

![Stock Kasten dashboard showing No data on every panel after import](/images/posts/2026-08-27-observability-grafana-multi-cluster-dashboard/grafana-unmodified-kasten-dashboard.png)

#### No cluster filter

None of the panel queries filters on `cluster_name`, because the dashboard was not intended to see more than one cluster. Import it against your central backend unchanged and you get one of two outcomes:

- Panels showing raw metrics, such as `catalog_persistent_volume_free_space_percent`, return blended series with no way to identify their cluster.
- Panels using `sum()` or `increase()`, such as actions failed in the last 24 hours, silently add metrics from every cluster into one figure.

Neither result looks obviously broken. That is why it is worth fixing before relying on the dashboard.

## Requirements

Before making any changes, make sure you have the following.

### Grafana

- A Grafana instance with permission to edit dashboards.
- The Prometheus-compatible backend from [Part 1]({% post_url 2026-05-06-observability-series-1-prometheus-remote-writes %}) added as a Grafana data source.

{% include note.html content="This guide uses Grafana Cloud's Prometheus data source for the examples and screenshots. Any Prometheus-compatible backend, including Thanos Receive, Cortex, Mimir, or a self-hosted service, works the same way once it has been added as a Grafana data source." %}

### Kasten dashboard

- The original [Kasten dashboard JSON](https://grafana.com/grafana/dashboards/21065-k10-dashboard/) if you want to make the variable and query changes yourself.
- Or the [modified dashboard JSON](https://gist.github.com/Jaiganeshjk12/27140f99a214a256ef6a1700cacce17c), ready to import directly.

{% include note.html content="The original dashboard's queries reference a data source named Prometheus by name, rather than through a picker. If you start from the original and do not have a data source with that exact name, panels show no data until you complete the templating step below. The modified version already uses ${datasource}." %}

## What This Guide Does Not Cover

To keep this part focused on dashboard customization, we are intentionally not covering:

- Installing or configuring Grafana itself.
- Adding a Prometheus-compatible backend as a Grafana data source.
- Alerting on these metrics, which is covered in Part 3.
- Building a Grafana dashboard from scratch.

This post retrofits the official Kasten dashboard for a multi-cluster metrics backend.

## Setup

### Import the modified dashboard

The fast path is to import the modified version of the [officially published Kasten dashboard](https://grafana.com/grafana/dashboards/21065-k10-dashboard/). It retains the same core metrics, adds a few useful panels, and already includes datasource and `cluster_name` variables.

1. In Grafana, go to **Dashboards -> New -> Import**.
2. Paste the [modified dashboard JSON](https://gist.github.com/Jaiganeshjk12/27140f99a214a256ef6a1700cacce17c), or upload it as a file.
![Grafana import dashboard](/images/posts/2026-08-27-observability-grafana-multi-cluster-dashboard/grafana-import-dashboard-json.png)
3. After import, the `datasource` dropdown defaults to the Grafana instance's default Prometheus-type data source. Make sure it is set to the Prometheus-compatible backend that holds your Kasten metrics from Part 1.
![Grafana Datasource variable dropdown](/images/posts/2026-08-27-observability-grafana-multi-cluster-dashboard/grafana-datasource-dropdown.png)
4. Use the `cluster_name` dropdown at the top of the dashboard to switch between clusters.
![Grafana cluster_name variable dropdown](/images/posts/2026-08-27-observability-grafana-multi-cluster-dashboard/grafana-cluster_name-variable-dropdown.png)

Every panel is already wired to `${datasource}` and filtered by `cluster_name`.

### What changed from the original dashboard

If you want to understand the changes, or adapt the dashboard for a differently named label, the modified version makes these updates:

- Every panel's data source, previously hardcoded to `Prometheus`, now references the `${datasource}` variable.
- Every query that returns per-cluster data filters on `cluster_name="$cluster_name"`. This is the label Part 1 attaches through the Helm chart's `clusterName` field.
- Two variables are added: `datasource`, a Datasource-type variable, and `cluster_name`, a Query-type variable chained to `${datasource}` with `label_values(action_ended_total, cluster_name)`.
- A few panels beyond the original dashboard are added, including a table view for a quick per-policy failure breakdown.

{% include note.html content="The modified dashboard is a community-maintained derivative of Kasten's published dashboard, not an official release. If Kasten adds panels or metrics to the original, update this version manually to include them." %}

## Verify

Once the dashboard is imported, verify that its variables control every panel correctly.

### Switch the data source

Change the `datasource` dropdown to another Prometheus-compatible backend, if one is available, and confirm that every panel repoints. If a panel still shows the old backend's data, its data source reference was not updated.

### Switch clusters

Use the `cluster_name` dropdown to move between real clusters and confirm that the displayed values change. If every cluster shows identical figures, the cluster filter is not applied to the panel queries.
##### Dashboard view selecting jai-monitoring cluster
![Grafana cluster selection jai-monitoring](/images/posts/2026-08-27-observability-grafana-multi-cluster-dashboard/grafana-jai-monitoring.png)
##### Dashboard view selecting ocp-support-lab cluster
![Grafana cluster selection ocp-support-lab](/images/posts/2026-08-27-observability-grafana-multi-cluster-dashboard/grafana-ocp-support-lab.png)

### Confirm the cluster list

The cluster_name dropdown should show your actual cluster names and nothing else. It obtains its values from: `label_values(action_ended_total, cluster_name)`

If expected clusters are missing, confirm in Grafana Explore that `action_ended_total` or any of the Kasten metrics include the `cluster_name` label — for example, with 

```
count by (cluster_name) (action_ended_total)
```
## Conclusion

You now have one dashboard that works no matter how many clusters feed into your metrics backend. Instead of assuming there's only ever one Kasten installation, each panel can be scoped to whichever cluster you select.

In **Part 3**, we will turn these metrics into alerts so you do not have to watch the dashboard to discover a backup problem.