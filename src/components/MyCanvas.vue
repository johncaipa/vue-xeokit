<template>
  <input type="checkbox" id="info-button"/>
  <label for="info-button" class="info-button"><i class="fa fa-info-circle" style="font-size:24px"></i></label>
  <canvas id="myCanvas"></canvas>
  <canvas id="myNavCubeCanvas"></canvas>
  <div id="treeViewContainer"></div>
  <div class="slideout-sidebar">
  <h1>Picking Objects</h1>
  <h2>Click objects to colorize them</h2>
    <div id="time">Loading JavaScript modules...</div>
    <p>ID Entidad: {{ entityId }}</p>
</div>
<p id="properties-text" v-if="entityId">
    GUID: 
    {{ entityId }} | Name: 
</p>
</template>

<script>
import {Viewer, WebIFCLoaderPlugin, XKTLoaderPlugin, NavCubePlugin, TreeViewPlugin} from 
"https://cdn.jsdelivr.net/npm/@xeokit/xeokit-sdk/dist/xeokit-sdk.es.min.js";

export default {
  data() {
    return {
      entityId: null,
    };
  },

  mounted() {
    //------------------------------------------------------------------------------------------------------------------
    // Create a Viewer, arrange the camera, tweak x-ray and highlight materials
    //------------------------------------------------------------------------------------------------------------------

    const viewer = new Viewer({
        canvasId: "myCanvas",
        transparent: true
    });

    const cameraControl = viewer.cameraControl;
    const scene = viewer.scene;
    const cameraFlight = viewer.cameraFlight;

    cameraControl.followPointer = true;
    cameraFlight.duration = 1.0;
    cameraFlight.fitFOV = 25;

    viewer.camera.eye = [-3.93, 2.85, 27.01];
    viewer.camera.look = [4.40, 3.72, 8.89];
    viewer.camera.up = [-0.01, 0.99, 0.039];

    viewer.scene.xrayMaterial.fillAlpha = 0.1;
    viewer.scene.xrayMaterial.fillColor = [0, 0, 0];
    viewer.scene.xrayMaterial.edgeAlpha = 0.4;
    viewer.scene.xrayMaterial.edgeColor = [0, 0, 0];

    viewer.scene.highlightMaterial.fill = false;
    viewer.scene.highlightMaterial.fillAlpha = 0.3;
    viewer.scene.highlightMaterial.edgeColor = [1, 1, 0];

    //------------------------------------------------------------------------------------------------------------------
    // Create a NavCube
    //------------------------------------------------------------------------------------------------------------------

    new NavCubePlugin(viewer, {
        canvasId: "myNavCubeCanvas",
        visible: true,
        size: 250,
        alignment: "bottomRight",
        bottomMargin: 100,
        rightMargin: 10
    });

    //------------------------------------------------------------------------------------------------------------------
    // Create an IFC structure tree view
    //------------------------------------------------------------------------------------------------------------------

    const treeView = new TreeViewPlugin(viewer, {
        containerElement: document.getElementById("treeViewContainer"),
        autoExpandDepth: 3, // Initially expand tree three storeys deep
        hierarchy: "containment" //containment,stories,types
    });

    //----------------------------------------------------------------------------------------------------------------------
    // Load a model and fit it to view
    //----------------------------------------------------------------------------------------------------------------------

    const webIFCLoader = new WebIFCLoaderPlugin(viewer, {
        // Path to web-ifc.wasm, which does the IFC parsing for us
        wasmPath: "https://cdn.jsdelivr.net/npm/@xeokit/xeokit-sdk/dist/"
    });

    const sceneModel = webIFCLoader.load({ // Returns an Entity that represents the model
        src: "models/ifc/Duplex.ifc",
        edges: true,
        excludeUnclassifiedObjects: false,
        //rotation: [0,0,0],
        //scale: [1, 1, 1],
        //origin: [10, 0, 0],
        //includeTypes: ["IfcWallStandardCase"],
        //excludeTypes: ["IfcSpace"],
    });

    // const xktLoader = new XKTLoaderPlugin(viewer);

    // const sceneModel = xktLoader.load({
    //     id: "Widget",
    //     src: "Schependomlaan.ifc.xkt",
    //     edges: true,
    //     excludeUnclassifiedObjects: false
    // });

    const t0 = performance.now();
    document.getElementById("time").innerHTML = "Loading model...";
    sceneModel.on("loaded", function () {
        const t1 = performance.now();
        document.getElementById("time").innerHTML = "Model loaded in " + Math.floor(t1 - t0) / 1000.0 + " seconds<br>Objects: " + sceneModel.numEntities;
    });

        //------------------------------------------------------------------------------------------------------------------
    // Mouse over entities to highlight them
    //------------------------------------------------------------------------------------------------------------------

    var hooverlastEntity = null;

    viewer.scene.input.on("mousemove", function (coords) {

        var hit = viewer.scene.pick({
            canvasPos: coords
        });

        if (hit) {

            if (!hooverlastEntity || hit.entity.id !== hooverlastEntity.id) {

                if (hooverlastEntity) {
                    hooverlastEntity.highlighted = false;
                }

                hooverlastEntity = hit.entity;
                hit.entity.highlighted = true;
            }
        } else {

            if (hooverlastEntity) {
                hooverlastEntity.highlighted = false;
                hooverlastEntity = null;
            }
        }
    });

    window.treeView = treeView;

    //------------------------------------------------------------------------------------------------------------------
    // Click Entities to colorize them
    //------------------------------------------------------------------------------------------------------------------

    var lastEntity = null;
    var lastColorize = null;

    viewer.cameraControl.on("picked", (pickResult) => {

        if (!pickResult.entity) {
            return;
        }
        var objectId = pickResult.entity.id;
        console.log(objectId);
        this.entityId = objectId;
        treeView.showNode(objectId);

        if (!lastEntity || pickResult.entity.id !== lastEntity.id) {

            if (lastEntity) {
                lastEntity.colorize = lastColorize;
            }

            lastEntity = pickResult.entity;
            lastColorize = pickResult.entity.colorize ? pickResult.entity.colorize.slice() : null;

            pickResult.entity.colorize = [0.0, 1.0, 0.0];
        }
    });

    viewer.cameraControl.on("pickedNothing", () => {
        if (lastEntity) {
            lastEntity.colorize = lastColorize;
            lastEntity = null;
        }
    });


  },
};
</script>
<style>

body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
    font-size: 1rem;
    font-weight: 400;
    line-height: 1.5;
    color: #212529;
    background-color: white;
    text-decoration: none;
    word-spacing: normal;
    text-align: left;
    letter-spacing: 0;
    -webkit-font-smoothing: antialiased;
    overflow-y: hidden;
    overflow-x: hidden;
    margin: 0;
    width: 100%;
    height: 100%;
    user-select: none;
}

#myCanvas {
    width: 100%;
    height: 600px;
    position: absolute;
    background: lightblue;
    background-image: linear-gradient(lightblue,white);
}

.slideout-sidebar {
    background: white;
    position: fixed;
    top: 30;
    right: -380px;
    z-index: 100000000;
    width: 350px;
    height: 100%;
    overflow-y: scroll;
    transition: all 300ms ease-in-out;
    border: 2px solid #212529;
    margin-top: 0;
    padding: 0px 10px 10px 20px;
}

.info-icon {
    width: 100px;
    height: auto;
    padding-top: 20px;
}

#info-button {
    display: none;
}

#info-button:hover {
    cursor: pointer;
}

.info-button {
    position: absolute;
    right: 15px;
    top: 30px;
    z-index: 1;
    transition: all 300ms ease-in-out;
}

.info-button:hover {
    cursor: pointer;
    color: black;
}

#info-button:checked ~ .slideout-sidebar {
    right: 0;
}

#info-button:checked + .info-button {
    right: 400px;
}

#myDatGuiContainer {
    margin-top: 20px;
    margin-bottom: 25px;
}

h1, h2, h3 {
    font-family: inherit;
    line-height: 1.2;
    color: inherit;
}

h1 {
    font-weight:500;
    margin-top: 0;
    margin-bottom: 0;
    padding-top: 15px;
    font-size: 1.4rem;
}

h1 a {
    color: #fffbfb;
}

h2 {
    font-size: 1.0rem;
    font-weight: 550;
    padding-top: 30px;
    padding-left: 0px;
    margin-top: 0px;
    padding-bottom: 10px;
    margin-bottom: 0;
    letter-spacing: 0px;
    /*border-bottom: 2px solid #d4d4d4;*/
}

h3 {
    font-size: 1.0rem;
    font-weight: 550;
    padding: 0;
    margin: 0;
    padding-left: 0px;
    padding-right: 40px;
    padding-top: 14px;
    padding-bottom: 8px;
}

.xeokit-camera-pivot-marker {
    color: #ffffff;
    position: absolute;
    width: 25px;
    height: 25px;
    border-radius: 15px;
    border: 2px solid #ebebeb;
    background: #4444;
    visibility: hidden;
    box-shadow: 5px 5px 15px 1px #212529;
    z-index: 10000;
    pointer-events: none;
}
  /* ----------------------------------------------------------------------------------------------------------*/
  /* NavCubePlugin */
  /* ----------------------------------------------------------------------------------------------------------*/

  #myNavCubeCanvas {
      position: absolute;
      width: 250px;
      height: 250px;
      bottom: 50px;
      right: 10px;
      z-index: 200000;
  }

  /* ----------------------------------------------------------------------------------------------------------*/
  /* TreeViewPlugin */
  /* ----------------------------------------------------------------------------------------------------------*/

  #treeViewContainer {
      pointer-events: all;
      height: 600px;
      overflow-y: scroll;
      overflow-x: scroll;
      position: absolute;
      background-color: rgba(255, 255, 255, 0.2);
      color: black;
      top: 30px;
      z-index: 200000;
      float: left;
      left: 0;
      padding-left: 10px;
      font-family: 'Roboto', sans-serif;
      font-size: 15px;
      user-select: none;
      -ms-user-select: none;
      -moz-user-select: none;
      -webkit-user-select: none;
      width: 350px;
      scrollbar-width: none; /* Firefox */
      -ms-overflow-style: none;  /* IE 10+ */
  }
  /* Hide scrollbar for Chrome, Safari and Opera */
  ::-webkit-scrollbar {
    display: none;
  }
  #treeViewContainer ul {
      list-style: none;
      padding-left: 1.75em;
      pointer-events: none;
  }

  #treeViewContainer ul li {
      position: relative;
      width: 200px;
      pointer-events: none;
      padding-top: 3px;
      padding-bottom: 3px;
      vertical-align: middle;
  }

  #treeViewContainer ul li a {
      background-color: #eee;
      border-radius: 50%;
      color: #000;
      display: inline-block;
      height: 1.5em;
      left: -1.5em;
      position: absolute;
      text-align: center;
      text-decoration: none;
      width: 1.5em;
      pointer-events: all;
  }

  #treeViewContainer ul li a.plus {
      background-color: #ded;
      pointer-events: all;
  }

  #treeViewContainer ul li a.minus {
      background-color: #eee;
      pointer-events: all;
  }

  #treeViewContainer ul li a:active {
      top: 1px;
      pointer-events: all;
  }

  #treeViewContainer ul li span:hover {
      color: white;
      cursor: pointer;
      background: black;
      padding-left: 2px;
      pointer-events: all;
  }

  #treeViewContainer ul li span {
      display: inline-block;
      width: calc(100% - 50px);
      padding-left: 2px;
      pointer-events: all;
      height: 23px;
  }

  #treeViewContainer .highlighted-node { /* Appearance of node highlighted with TreeViewPlugin#showNode() */
      border: black solid 1px;
      background: yellow;
      color: black;
      padding-left: 1px;
      padding-right: 5px;
      pointer-events: all;
  }
  #properties-text {
    position: absolute;
    left: 0%;
    bottom: 0%;
    z-index: 100;
}
</style>