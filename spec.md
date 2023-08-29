# Matrix Whiteboard (TheBoard)
This defines a set of events which describe structures in a 2d plane.
These events are shared over a matrix room with all participants.
They can interactively create content on a whiteboard.

## Draw Events
### Path
A path event can consist of multiple paths and a position. 

**Batching:** Events fired rapidly after each other should be batched and send as one event with multiple paths (examples: `:`, `ä` or quickly written words like `and` can all be sent as one event even though they consist of multiple paths)

**Segments:** Segments consist of sets of two or six elements:

Two: `[Pos.x, pos.y]`

Six: `[Pos.x, pos.y, HandleStart.x,HandleStart.y, HandleEnd.x, HandleEnd.y]`

**Position** The position is the top left corner of the path. The segment positions are relative to `postition`.

**Fill Color** The `fillColor` is a hex formatted color. No infill with `#00000000` alpha channel.

**Stroke Color** The `strokeColor` is a hex formatted color.

**Stroke Width** The `strokeWidth` is a `float`. Can be set to `0` to only draw the infill.

**Pressure Sensitive (var. thickness) Strokes:** Are achievable by calculating the contour of the whole stroke with segments. Than the `fillColor` is used as the stroke color and the `strokeWidth` is `0.0`. For pressure sensitive strokes that have infill (actual infill not the "fake" infill required for the variable stroke thickness) two paths are sent.
```JSON
"com.github.TheBoard.object": {
    "content": {
        "version":NUMBER,
        "objtype":"path",
        "paths": [
            {
                "segments": [
                    ["0.0 0.0 0.0 0.0 0.0 0.0"],[]
                ],
                "closed":BOOL
                "fillColor": "",
                "strokeColor": "",
                "strokeWidth": 0.0,
                "position": {
                    "x": 0.0,
                    "y": 0.0
                }
            }
        ],
        
    }
},
```

### Text

```JSON
"com.github.TheBoard.object": {
    "content": {
        "version":3,
        "text": "Hello World",
        "fontSize": 20,
        "fontFamily": "",
        "color":"#000",
        "position": {
            "x": 100.0,
            "y": 0.0
        },
        "objtype":"text"
    }
}
```
### Image
The image file itself is uploaded to a matrix file server. The event only stores the URL (similar to `"m.room.message"` and `"msgtype":"m.image"`).
```JSON
"com.github.TheBoard.object": {
    "content": {
        "url": "",
        "size": {
            "width": 0.0,
            "height": 0.0
        },
        "position": {
            "x": 0.0,
            "y": 0.0
        },
        "objtype":"image"
    }
}
```
### Pdf's
The whole Pdf is uploaded. All interesting pages then get positioned with this event and rendered per page on the canvas. 

The position and size will (in most scenarios) be generated by the client. So the user does not need to manually position each page.
```JSON
"com.github.TheBoard.object": {
    "content": {
        "url": "",
        "objtype":"pdf",
        "pages": [
            {
                "pdfPageIndex":0,
                "position": {
                    "x": 0.0,
                    "y": 0.0
                },
                "size": {
                    "width": 0.0,
                    "height": 0.0
                }
            }
        ]
    }
}
```

## Pages
This is a structure feature of TheBoard. A page is just a frame/box. It is only rendered as a guide. 

**Structure:**
Multiple pages can be aligned on the whiteboard. A use-case example would be to do different exercised on different pages.

**Exporting:**
The main purpose is exporting. When exporting as a pdf the user can select a collection of pages and they will than be converted to a pdf. A unlimited 2d canvas is otherwise hard to convert to a 1d array of pages.

**Grid:** Another option is to provide custom grids for each page. This is how line or square grids become available.
```JSON
"com.github.TheBoard.page": {
    "content": {
        "position": {
            "x": 0.0,
            "y": 0.0
        },
        "size": {
            "width": 0.0,
            "height": 0.0
        },
        "grid":"",
        "dinSize":"",
        "label":""
    }
}
```

## Edits and redacts
Since there is no need/advantage to know the previous state, there are no edits with the `m.replace` relation. Instead, the edited message is redacted and a new message is sent. This makes it really simple because the whiteboard is just the collection of non-redacted events.
## Commits
This event exists for loading performance reasons. After a defined amount of events or on user request, a commit can be sent. This commit contains all **non-redacted** events to date. They are stored and compressed into a file (called a `commitFile`) and uploaded to a matrix file server.
```JSON
"com.github.TheBoard.commit": {
    "content": {
        "url":""
    }
}
```
From the commit event, the following information can be obtained:
 - date of the commit
 - the `commitFile` URL to get all events and draw the board.

To immediately have access to the commit a room state event is sent.
```JSON
"com.github.TheBoard":{
    "lastCommit":"commitID"
}
```
Loading is then done by 
 1. getting the commitID from the room state.
 1. downloading the commitFile
 1. parsing the commitFile
 1. do a scroll-back on the room.time-line until the time stamp of the commit event.
 1. during the scroll-back remove the redacted events from the parsed commitFile and add the new events.
### Commit File
The commit file is just a json:
 - the fields: `room_id`, `unsigned`, `event_id` (, `type`) could removed from EVENT.
```JSON
{
    "event_id1":{
        EVENT
    },
    "event_id2":{
        EVENT
    }
}
```
## Room State
A matrix room is parsed as a whiteboard with the `com.github.TheBoard` state key. It contains the settings and the `lastCommit`.
```JSON
"com.github.TheBoard":{
    "lastCommit":"commitID",
    "settings":{
        "colorpalette":["color"],
        "layers":["roomId"],
        "isLayer":false
    }
}
```
### Last Commit
See section **Commits**
### settings.colorpalette
Each room can have its own color palette. It is just an array of colors. A darker variant of each color will be calculated and displayed automatically. There is good usability for up to 10 colors. More than 10 colors are not recommended.
### settings.layers
A room can have layers. Layers are other rooms that are not listed in the notebook/whiteboard list. A layer can be hidden or shown. The big feature that comes with layer is that you can make non destructive modification to a whiteboard with DIFFERENT ACCESS RULES than the underlying whiteboard/layer. This can be used for private comments on a bigger shared document.

### settings.isLayers
When a room is marked as a layer it is not listed in the notebook/whiteboard list. This flag is redundant (a layer can also be determined by checking the `settings.layers` array from all other rooms).