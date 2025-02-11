
![](attachments/Screenshot%202025-02-04%20at%2016.07.12.png)

Below is an example of a “vanilla‐JS” library that creates an infinite canvas with draggable nodes and connectors. You can include the library in your project and use its API to add nodes, drag them around, and connect their “input” and “output” points. (In this example a node has one input and one output, but you can easily extend it.) The code uses plain JavaScript (no TypeScript or external libraries) and uses an SVG overlay for drawing connector lines.

Below you’ll find two files:

1. **vanillaFlow.js** – the library code.
2. **index.html** – an example HTML page that loads the library, some basic CSS, and shows how to create nodes and use the API.

---

### vanillaFlow.js

```js
(function (global) {
  // Constructor for the VanillaFlow library
  function VanillaFlow(container) {
    // The container element should be position relative and overflow hidden.
    this.container = container;
    this.nodes = [];         // array of node objects
    this.connections = [];   // array of connection objects

    // Create an SVG overlay to draw connections.
    this.svg = document.createElementNS("http://www.w3.org/2000/svg", "svg");
    this.svg.style.position = "absolute";
    this.svg.style.top = "0";
    this.svg.style.left = "0";
    this.svg.style.width = "100%";
    this.svg.style.height = "100%";
    this.svg.style.pointerEvents = "none"; // allow clicks to go through
    container.appendChild(this.svg);

    // State for dragging nodes and drawing connectors.
    this.currentNode = null;
    this.offsetX = 0;
    this.offsetY = 0;
    this.isDraggingNode = false;

    // State for connection drawing.
    this.isDrawingConnection = false;
    this.connectionStart = null; // {node, type, element}
    this.tempLine = null;

    // Bind event handlers
    this._bindCanvasEvents();
  }

  // Create a new node at (x, y). Options may include label text.
  VanillaFlow.prototype.addNode = function (x, y, options) {
    options = options || {};
    var node = document.createElement("div");
    node.className = "vf-node";
    node.style.position = "absolute";
    node.style.left = x + "px";
    node.style.top = y + "px";
    node.style.cursor = "move";
    node.innerHTML = options.label || "Node";

    // Create an input connector.
    var inputConnector = document.createElement("div");
    inputConnector.className = "vf-connector input";
    // position on the left center of the node.
    inputConnector.style.position = "absolute";
    inputConnector.style.width = "10px";
    inputConnector.style.height = "10px";
    inputConnector.style.borderRadius = "50%";
    inputConnector.style.background = "#3498db";
    inputConnector.style.top = "50%";
    inputConnector.style.left = "-5px";
    inputConnector.style.transform = "translateY(-50%)";
    // Mark type for later reference.
    inputConnector.dataset.type = "input";

    // Create an output connector.
    var outputConnector = document.createElement("div");
    outputConnector.className = "vf-connector output";
    // position on the right center of the node.
    outputConnector.style.position = "absolute";
    outputConnector.style.width = "10px";
    outputConnector.style.height = "10px";
    outputConnector.style.borderRadius = "50%";
    outputConnector.style.background = "#e74c3c";
    outputConnector.style.top = "50%";
    outputConnector.style.right = "-5px";
    outputConnector.style.transform = "translateY(-50%)";
    outputConnector.dataset.type = "output";

    // Append connectors to the node.
    node.appendChild(inputConnector);
    node.appendChild(outputConnector);

    // Append node to container.
    this.container.appendChild(node);

    // Save node info.
    var nodeObj = {
      el: node,
      input: inputConnector,
      output: outputConnector,
      x: x,
      y: y,
      width: node.offsetWidth,
      height: node.offsetHeight
    };
    this.nodes.push(nodeObj);

    // Bind events for node dragging.
    this._bindNodeEvents(nodeObj);
    // Bind events for connector drawing.
    this._bindConnectorEvents(nodeObj);

    return nodeObj;
  };

  // Binds mouse events for dragging nodes.
  VanillaFlow.prototype._bindNodeEvents = function (nodeObj) {
    var self = this;
    nodeObj.el.addEventListener("mousedown", function (e) {
      // Only start dragging if the target is the node itself, not a connector.
      if (e.target.classList.contains("vf-connector")) return;
      self.isDraggingNode = true;
      self.currentNode = nodeObj;
      // Calculate offset between mouse and node top-left.
      var rect = nodeObj.el.getBoundingClientRect();
      self.offsetX = e.clientX - rect.left;
      self.offsetY = e.clientY - rect.top;
      e.stopPropagation();
    });
  };

  // Binds mouse events for drawing connections from connectors.
  VanillaFlow.prototype._bindConnectorEvents = function (nodeObj) {
    var self = this;
    // Both input and output connectors get a mousedown handler.
    [nodeObj.input, nodeObj.output].forEach(function (connEl) {
      connEl.addEventListener("mousedown", function (e) {
        // We only allow drawing from an output connector.
        if (connEl.dataset.type !== "output") return;
        self.isDrawingConnection = true;
        self.connectionStart = { node: nodeObj, connector: connEl };
        // Create a temporary SVG line element.
        self.tempLine = document.createElementNS("http://www.w3.org/2000/svg", "line");
        self.tempLine.setAttribute("stroke", "#2c3e50");
        self.tempLine.setAttribute("stroke-width", "2");
        self.svg.appendChild(self.tempLine);
        // Update the line’s starting position.
        var pos = self._getConnectorCenter(connEl, nodeObj.el);
        self.tempLine.setAttribute("x1", pos.x);
        self.tempLine.setAttribute("y1", pos.y);
        self.tempLine.setAttribute("x2", pos.x);
        self.tempLine.setAttribute("y2", pos.y);
        e.stopPropagation();
      });
    });
  };

  // Binds events for the canvas (for dragging nodes and drawing connections).
  VanillaFlow.prototype._bindCanvasEvents = function () {
    var self = this;
    // Mouse move on document to update node dragging and connector drawing.
    document.addEventListener("mousemove", function (e) {
      // Drag node if active.
      if (self.isDraggingNode && self.currentNode) {
        var containerRect = self.container.getBoundingClientRect();
        var newX = e.clientX - containerRect.left - self.offsetX;
        var newY = e.clientY - containerRect.top - self.offsetY;
        self.currentNode.x = newX;
        self.currentNode.y = newY;
        self.currentNode.el.style.left = newX + "px";
        self.currentNode.el.style.top = newY + "px";
        // Update connections that involve this node.
        self._updateConnections();
      }
      // If drawing a connection, update the temporary line.
      if (self.isDrawingConnection && self.tempLine && self.connectionStart) {
        var startPos = self._getConnectorCenter(self.connectionStart.connector, self.connectionStart.node.el);
        self.tempLine.setAttribute("x1", startPos.x);
        self.tempLine.setAttribute("y1", startPos.y);
        self.tempLine.setAttribute("x2", e.clientX);
        self.tempLine.setAttribute("y2", e.clientY);
      }
    });

    // Mouse up anywhere on document stops node dragging and (possibly) finishes a connection.
    document.addEventListener("mouseup", function (e) {
      // Stop node dragging.
      self.isDraggingNode = false;
      self.currentNode = null;

      // Handle connection drawing end.
      if (self.isDrawingConnection) {
        // Try to see if we ended on an input connector.
        var target = e.target;
        if (target.classList.contains("vf-connector") && target.dataset.type === "input") {
          // Find the node object that owns this connector.
          var targetNode = self.nodes.find(function (nodeObj) {
            return nodeObj.input === target;
          });
          if (targetNode) {
            // Finalize the connection.
            self._finalizeConnection(self.connectionStart.node, self.connectionStart.connector, targetNode, target);
          }
        }
        // Remove the temporary line.
        if (self.tempLine && self.tempLine.parentNode) {
          self.svg.removeChild(self.tempLine);
        }
        self.tempLine = null;
        self.isDrawingConnection = false;
        self.connectionStart = null;
      }
    });
  };

  // Returns the center point (in page coordinates) of a connector element.
  VanillaFlow.prototype._getConnectorCenter = function (connectorEl, nodeEl) {
    // Get the bounding rectangle of the connector.
    var rect = connectorEl.getBoundingClientRect();
    return { x: rect.left + rect.width / 2, y: rect.top + rect.height / 2 };
  };

  // Finalizes a connection from an output to an input.
  VanillaFlow.prototype._finalizeConnection = function (fromNode, fromConnector, toNode, toConnector) {
    var self = this;
    // Create a permanent SVG line.
    var line = document.createElementNS("http://www.w3.org/2000/svg", "line");
    line.setAttribute("stroke", "#2c3e50");
    line.setAttribute("stroke-width", "2");
    self.svg.appendChild(line);

    // Store connection data.
    var connection = {
      from: { node: fromNode, connector: fromConnector },
      to: { node: toNode, connector: toConnector },
      line: line
    };
    self.connections.push(connection);

    // Update the line’s position.
    self._updateConnectionLine(connection);
  };

  // Updates a single connection line (called when nodes move).
  VanillaFlow.prototype._updateConnectionLine = function (connection) {
    var fromPos = this._getConnectorCenter(connection.from.connector, connection.from.node.el);
    var toPos = this._getConnectorCenter(connection.to.connector, connection.to.node.el);
    connection.line.setAttribute("x1", fromPos.x);
    connection.line.setAttribute("y1", fromPos.y);
    connection.line.setAttribute("x2", toPos.x);
    connection.line.setAttribute("y2", toPos.y);
  };

  // Updates all connection lines.
  VanillaFlow.prototype._updateConnections = function () {
    var self = this;
    self.connections.forEach(function (conn) {
      self._updateConnectionLine(conn);
    });
  };

  // Expose the VanillaFlow class
  global.VanillaFlow = VanillaFlow;
})(window);
```

---

### index.html

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>VanillaFlow Demo</title>
  <style>
    /* Basic styles for the canvas and nodes */
    body, html {
      margin: 0;
      padding: 0;
      height: 100%;
      overflow: hidden;
      background: #ecf0f1;
    }
    #canvas {
      position: relative;
      width: 100%;
      height: 100%;
      cursor: default;
      /* Optionally, set a grid background for an “infinite” feel */
      background-image: linear-gradient(0deg, transparent 24%, rgba(0,0,0,0.05) 25%, rgba(0,0,0,0.05) 26%, transparent 27%, transparent 74%, rgba(0,0,0,0.05) 75%, rgba(0,0,0,0.05) 76%, transparent 77%, transparent), 
                        linear-gradient(90deg, transparent 24%, rgba(0,0,0,0.05) 25%, rgba(0,0,0,0.05) 26%, transparent 27%, transparent 74%, rgba(0,0,0,0.05) 75%, rgba(0,0,0,0.05) 76%, transparent 77%, transparent);
      background-size: 50px 50px;
    }
    .vf-node {
      padding: 10px 20px;
      background: white;
      border: 1px solid #bdc3c7;
      border-radius: 4px;
      box-shadow: 2px 2px 5px rgba(0,0,0,0.1);
      user-select: none;
    }
    .vf-connector {
      width: 10px;
      height: 10px;
      border: 2px solid white;
      box-sizing: border-box;
      cursor: pointer;
    }
    /* Different colors for input and output connectors */
    .vf-connector.input {
      background: #3498db;
    }
    .vf-connector.output {
      background: #e74c3c;
    }
  </style>
</head>
<body>
  <div id="canvas"></div>

  <!-- Include the library -->
  <script src="vanillaFlow.js"></script>
  <script>
    // Create an instance of VanillaFlow using the canvas element.
    var canvas = document.getElementById("canvas");
    var flow = new VanillaFlow(canvas);

    // For demonstration, add a few nodes.
    flow.addNode(100, 100, { label: "Node A" });
    flow.addNode(300, 200, { label: "Node B" });
    flow.addNode(500, 150, { label: "Node C" });

    // (Optional) You could add UI controls that let the user create new nodes dynamically.
    // For example, clicking on the canvas:
    canvas.addEventListener("dblclick", function(e) {
      // Calculate position relative to the canvas.
      var rect = canvas.getBoundingClientRect();
      var x = e.clientX - rect.left;
      var y = e.clientY - rect.top;
      flow.addNode(x, y, { label: "New Node" });
    });
  </script>
</body>
</html>
```

---

### How It Works

- **Canvas & Nodes:**  
    The library expects a container element (with id `"canvas"` in our example) that acts as the infinite canvas. Nodes are absolutely positioned `<div>` elements added to this container. Each node contains two “connector” `<div>` elements (one input and one output).
    
- **Dragging Nodes:**  
    When you mousedown on a node (but not on a connector), the node becomes draggable. Its position is updated as you move the mouse, and any connector lines attached to it update automatically.
    
- **Drawing Connectors:**  
    When you mousedown on an output connector (the red circle), a temporary SVG line follows your mouse. If you release (mouseup) over an input connector (the blue circle) of another node, the temporary line is replaced by a permanent connection line drawn in the SVG overlay. Multiple connections can be made.
    
- **Infinite Canvas:**  
    In this example the canvas “infinite” feel is achieved via a repeating grid background. (More advanced panning and zooming can be added if desired.)
    

You can extend this library by adding features like deletion of nodes/connections, custom node types, and panning/zooming of the canvas.

This implementation is kept simple for demonstration and can serve as a starting point for a more complete drag‐and‐drop flow editor similar to React Flow. Enjoy!