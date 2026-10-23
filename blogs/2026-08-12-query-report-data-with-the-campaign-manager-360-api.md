---
title: "Query report data with the Campaign Manager 360 API"
url: "http://ads-developers.googleblog.com/2026/08/query-report-data-with-campaign-manager.html"
date: "2026-08-12"
author: "Google Ads Developer Advisor"
feed_url: "http://ads-developers.googleblog.com/feeds/posts/default"
---
You can now use the reportData.query endpoint in the Campaign Manager 360 API to synchronously query your campaign performance data and retrieve structured JSON data directly in the response. The new reportData.query endpoint offers a simplified option for report data retrieval compared to the standard reporting API workflow : Queries run synchronously with a 60-second execution limit, making this endpoint ideal for real-time dashboards and ad-hoc data explorations. Report data is returned as structured JSON directly in the HTTP response, which eliminates file management overhead.
