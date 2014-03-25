# WFS Layer Plugin

<table class="table">
  <tr>
    <td>API</td><td>[link](<%= apiurl %>Oskari.mapframework.bundle.mapwfs.plugin.wfslayer.WfsLayerPlugin.html)</td>
  </tr>
</table>

## Description

WfsLayerPlugin does WFS-related queries, does featureinfo requests, draws image tiles and highlights features.

## Screenshot

![screenshot](/images/bundles/wfslayer.png)

## Bundle configuration

No configuration is required.

## Requests the plugin handles

This plugin doesn't handle any requests.

## Requests the plugin sends out

This plugin doesn't sends out any requests

## Events the bundle listens to

<table class="table">
  <tr>
    <th> Event </th><th> How does the bundle react</th>
  </tr>
  <tr>
    <td> EscPressedEvent </td><td> Closing GetInfo "popup" from screen.</td>
  </tr>
  <tr>
    <td> MapClickedEvent </td><td> Send ajax request to backend system.</td>
  </tr>
  <tr>
    <td> AfterMapMoveEvent </td><td> Cancel ajax request.</td>
  </tr>
</table>

## Events the plugin sends out

This bundle doesn't send any events.

## Dependencies

<table class="table">
  <tr>
    <th>Dependency</th><th>Linked from</th><th>Purpose</th>
  </tr>
  <tr>
    <td> [jQuery](http://api.jquery.com/) </td>
    <td> Version 1.7.1 assumed to be linked (on page locally in portal) </td>
    <td> Used to create the UI</td>
  </tr>
  <tr>
    <td> [Oskari infobox](<%= docsurl %>framework/infobox.html) </td>
    <td> Oskari's InfoBoxBundle </td>
    <td> That handles the infobox as an Openlayers popup with customized UI</td>
  </tr>
  <tr>
    <td> [Backend API](<%= docsurl %>backend/mapmodule/getinfoplugin.html) </td>
    <td> N/A </td>
    <td> Get info is handle in backend</td>
  </tr>
</table>
