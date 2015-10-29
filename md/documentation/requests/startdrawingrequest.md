# StartDrawingRequest

This request is used to activate a draw control on the map which allows the user to draw on the map.

## Parameters

(* means the parameter is required)

<table class="table">
<tr>
  <th> Name</th><th> Type</th><th> Description</th><th> Default value</th>
</tr>
<tr>
  <td>*id</td><td> String</td><td> Identifier for request</td><td> </td>
</tr>
<tr>
  <td>*shape</td><td> String</td><td> Drawing shape: Point/Circle/Polygon/Box/Square/LineString</td><td> </td>
</tr>
<tr>
  <td> {options}</td><td> Object</td><td> Parameters for options-object is listed below</td><td> null</td>
</tr>
<tr>
  <td> buffer</td><td> Number</td><td> Buffer for drawing buffered line and dot. If not given or 0, will disable dragging.</td><td> null</td>
</tr>
<tr>
  <td> style</td> <td> Object</td> <td> Styles for draw, modify and intersect mode. If options don't include custom style, sets default styles. See styling example at the last example of this page.</td>
  <td> 
  		{
    		fillColor: 'rgba(255,0,255,0.2)',
    		strokeColor: 'rgba(0,0,0,1)',
    		width: 2,
    		radius: 4,
    		lineDash: [5],
    		textScale: 1.3,
    		textOutlineColor: 'rgba(255,255,255,1)',
    		textColor: 'rgba(0,0,0,1)'
    	}
    </td>
</tr>
<tr>
  <td> allowMiltipleDrawing</td><td> Boolean/String</td><td> true - multiple selection is allowed.<br> false - after drawing is finished (by doubleclick), will stop drawing tool, but keeps selection on the map.<br> 'single' - selection will be removed before drawing a new selection.</td><td> true</td>
</tr>
<tr>
  <td> drawControl</td><td> Boolean</td><td> true - activates draw control.<br> false - drawing will not activated.</td><td> true</td>
</tr>
<tr>
  <td> modifyControl</td><td> Boolean</td><td> true - activates modify control.<br> false - modifying will not activated.</td><td> true</td>
</tr>
<tr>
  <td> showMeasure</td><td> Boolean</td><td> Use this parameter for displaying measurement result on line or polygon.<br> true - will display measure on selection.</td><td> false</td>
</tr>
<tr>
  <td> geojson</td><td> String</td> <td> Geojson for editing. If not given, will activate draw/modify control according to given shape.</td><td> null</td>
</tr>
</table>

For example:
```javascript
var sb = Oskari.getSandbox();
sb.postRequestByName('DrawTools.StartDrawingRequest', [
		id: 'bufferedLineSelection', 
		shape: 'LineString', 
		{
        	buffer: 200,
            allowMultipleDrawing: false,
			drawControl: true, 
            modifyControl: false
        }		
);
```

## Examples

Start to draw for 'measure' functionality and keep the drawing on the map:
```javascript
var sb = Oskari.getSandbox();
sb.postRequestByName('DrawTools.StartDrawingRequest', ['measure', 'LineString'], {
	showMeasure: true
});
```

After the drawing is completed a 'DrawingEvent' is triggered where id is 'measure' with the measure data available in event.getData().

Add a buffer of 50 units (metric) for the line shape we got from the previous 'measure' event:
```javascript
var sb = Oskari.getSandbox();
sb.postRequestByName('DrawTools.StartDrawingRequest', ['measure', 'LineString'], {
	buffer: 50
});
```

Start to draw a circle for 'myplaces' functionality (disabling the buffer by dragging feature and don't setup modify control after finishing):
```javascript
var sb = Oskari.getSandbox();
sb.postRequestByName('DrawTools.StartDrawingRequest', ['myplaces', 'Circle', {
    buffer : 50,
    modifyControl : false
}]);
```
Again on completion a 'DrawingEvent' is triggered with id 'myplaces' with the drawn shape as geojson available in event.getGeoJson().

Start to draw a polygon for 'myplaces' functionality with specific style and showing a label next to the drawing:
```javascript
var style = {
	draw : {
		fill : {
			 color: 'rgba(238,0,0,0.2)' 
		},
		stroke : {
    	      color: 'rgba(0,0,0,1)',
    	      width: 2,
    	      lineDash: 5
		},
    	image : {
    	      radius: 4,
    	      fill: {
    	        color: 'rgba(0,0,0,1)'
    	      }
    	},
    	text: {
    		scale: 1.3,
            fill: {
            	color: 'rgba(255,255,255,1)'
            },
            stroke: {
               color: 'rgba(0,0,0,1)',
               width: 2
            }
    	}
	}, 
	modify : {
		fill : {
			 color: 'rgba(153,102,255,0.2)' 
		},
		stroke : {
    	      color: 'rgba(0,0,0,1)',
    	      width: 2
    	},
    	image : {
    	      radius: 4,
    	      fill: {
    	        color: 'rgba(0,0,0,1)'
    	      }
    	},
    	text: {
    		scale: 1.3,
            fill: {
            	color: 'rgba(255,255,255,1)'
            },
            stroke: {
               color: 'rgba(0,0,0,1)',
               width: 2
            }
    	}
	},
	intersect : {
		fill : {
			 color: 'rgba(101,255,102,0.2)' 
		},
		stroke : {
    	      color: 'rgba(0,0,0,1)',
    	      width: 2,
    	      lineDash: 5
    	},
    	image : {
    	      radius: 4,
    	      fill: {
    	        color: 'rgba(0,0,0,1)'
    	      }
    	},
    	text: {
    		scale: 1.3,
            fill: {
            	color: 'rgba(255,255,255,1)'
            },
            stroke: {
               color: 'rgba(0,0,0,1)',
               width: 2
            }
    	}
	}
};

var sb = Oskari.getSandbox();
sb.postRequestByName('DrawTools.StartDrawingRequest', ['myplaces', 'Polygon', {
    label : 'My drawing',
    style : style
}]);
```
Again on completion a 'DrawingEvent' is triggered with id 'myplaces' with the drawn shape as geojson available in event.getGeoJson().
