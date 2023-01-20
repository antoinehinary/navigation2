# Nav2 Route Server

The Route Server is a Nav2 Task server to compliment the Planner Server's free-space planning capabilities with pre-defined Navigation Route Graph planning, created by [Steve Macenski](https://www.linkedin.com/in/steve-macenski-41a985101/) while at [Samsung Research America](https://www.sra.samsung.com/).
This graph has few rules associated with it and may be generated manually or automatically via AI, geometric, or probablistic techniques.
This package then takes a planning request and uses this graph to find a valid route through the environment via an optimal search-based algorithm.
It may also live monitor and analyze the route's process to execute custom behaviors on entering or leaving edges or achieving particular graph nodes.

There are plugin interfaces throughout the server to enable a great deal of application-specific customization:
- Custom search-prioritization behavior with edge scoring plugins (e.g. minimize distance or time, mark blocked routes, enact static or dynamic penalties for danger and application-specific knowledge, etc)
- Custom operations to perform during route execution, triggered when entering or leaving an edge or achieving a graph node on the path (e.g. open door, pause at node to wait for clearance, adjust maximum speed, etc)
- Parsers of navigation graph files to use any type of format desirable (e.g. geoJSON, OpenStreetMap)

Additionally, the parsers store **additional arbitrary metadata** specified within the navigation graph files to store information such as speed limits, added costs, or operations to perform.
Thus, we do not restrict the data that can be embedded in the navigation route graph for an application and this metadata is communicated to the edge scoring and operations plugins to adjust behavior.
Note however that plugins may also use outside information from topics, services, and actions for dynamic behavior or centralized knowledge sharing as well. 

## Features

- Optimized Dikjstra's planning algorithm modeled off of the Smac Planner A* implementation
- Cleverly designed for no run-time lookups on the graph during search (2 lookups to find the start and goal edges on request initialization)
- Use of Kd-trees for finding the nearest node to arbitrary start and goal poses in the graph
- Highly efficient graph representation to maximize caching in a single data structure containing both edges' and nodes' objects and relationships with localized information
- All edges are directional
- Data in files may be with respect to any frame in the TF tree and are transformed to a centralized frame automatically
- Operation interface response returns both a sparse route of nodes and edges for client applications with navigation graph knowledge and `nav_msgs/Path` dense paths minimicking freespace planning for drop-in behavior replacement of the Planner Server.  
- Operation interface request can process requests with start / goal node IDs or poses
- Service interface to change navigation route graphs at run-time
- Edge scoring dynamic plugins return a cost for traversing an edge and may mark an edge as invalid in current conditions
- Common edge scoring plugins are provided for needs like optimizing for distance, time, cost, static or dynamic penalties
- Graph file parsing dynamic plugins allow for use of custom or proprietary formats
- Provided 2 file parsing plugins based on popular standards: OpenStreetMap and GeoJSON
- Operation dynamic plugins to perform arbitrary tasks at a given node or when entering or leaving an edge on the route (plan)
- Operation may be graph-centric (e.g. graph file identifies operation to perform) or plugin-centric (e.g. plugins self-identify nodes and edges to act upon) 
- TODO Operations provided (needs Operations implemented)

## Design

TODO architectural diagram (needs final architecture hashed out)
TODO main elements of the package and their role (needs final architecture hashed out)

### Plugin Interfaces

TODO provide specifications for plugins // what they do // links (needs plugin API stability)

TODO Operations provide metadata but also provide metadata about nodes/edges as well

## Metrics

TODO provide analysis (needs completion)

The use of Kd-trees to find the nearest start and goal nodes in the graph to the request is over 140x faster than the brute force data-structure lookup (0.35 ms/1000 lookups vs 50 ms/1000 lookups).

## Parameters

TODO provide full list (needs completion)

## File Formats

Parsers are provided for OpenStreetMap (OSM) and GeoJSON formats.
The graphs may be stored in one of the formats the parser plugins can understand or implement your own parser for a particular format of your interest!

The only two required features of the navigation graph is for the nodes and edges to have identifiers from each other to be unique for referencing and for edges to have the IDs of the nodes belonging to the start and end of the edge.
This is strictly required for the Route Server to operate properly in all of its features.

Besides this, we establish some requirements and conventions for route graph files _for the provided parsers_ to standardize this information.
These conventions are utilized by the provided parsers to offer consistent and documented behavior. 
These are also special values stored in the graph nodes and edges outside of the arbitrarily defined metadata.

In the provided parsers, the unique identifier for each node and edge should be given as `id`. The edge's nodes are `startid` and `endid`.
While technically optional, it is highly recommended to also provide:
- The node's location and frame of reference (`x`, `y`, `frame`)
- The Operation's `trigger` (e.g. enter, exit edge, node achieved) and `type` (e.g. action to perform)

While optional, it is somewhat recommended to provide, if relevent:
- The edge's `cost`, if it is fixed or edge scoring plugins are not used
- Whether the edge's cost is `overridable` with edge scoring plugins, if provided

Otherwise, the Node, Edge, and Operations may contain other arbitrary application-specific fields with key-value pairs.
The keys are stored as strings in `metadata` within the objects which acts as a python3 dict (e.g. `std::unordered_map<std::string, std::any>`) with an accessor function `T getValue(const std::string & key, T & default_val)` to obtain the field's values within all plugins.

While neither OSM nor GeoJSON are YAML-based, the following YAML file is provided as a more human-readable example for illustration of the conventions above.
Example files in both formats can be found TODO LINKS TO EXAMPLE FILES

```
example_graph.yaml

Node1:                  // <-- If provided by format, store as name in metadata
  id: 1                 // <-- Required
  x: 0.32               // <-- Highly recommended
  y: 4.3                // <-- Highly recommended
  frame: "map"          // <-- Highly recommended
  location: "workshop"  // <-- Metadata for node (arbitrary)
  operation:
    pause:              // <-- If provided by format, store as name in metadata
      type: "stop"      // <-- Required
      trigger: ON_ENTER // <-- Required
      wait_for: 5.0     // <-- Metadata for operation (arbitrary)

Edge1:                  // <-- If provided by format, store as name in metadata
  id: 2                 // <-- Required
  startid: 1            // <-- Required
  endid: 3              // <-- Required
  speed_limit: 0.85     // <-- Metadata for edge (arbitrary)
  overridable: False    // <-- Recommended
  cost: 6.0             // <-- Recommended, if relevent
  operations:
    open_door:          // <-- If provided by format, store as name in metadata
      type: "open_door" // <-- Required
      trigger: ON_EXIT  // <-- Required
      door_id: 54       // <-- metadata for operation (arbirary)
```

### Metadata Conventions for Convenience

While other metadata fields are not required nor necessarily needed, there are some useful standards which may make your life easier within in the Route Server framework.
These are default fields which if used can make your life easier with the provided plugins, but you're free to embed this information anyway you like (but may come at the cost of needing to re-implement provided capabilities).

The `DistanceScorer` edge scoring plugin will come the L2 norm between the poses of the nodes constituting the edge as its unweighted score.
This uses the node's location information as "highly recommended" above.
It contains a parameter `speed_tag` (Default: `speed_limit`) which will check `edge.metadata` if it contains an entry for speed limit information, represented as a percentage of the maximum speed.
If so, it will adjust the score to instead be proportional to the time rather than distance.
Thus, by convention, we say that `speed_limit` attribute of an edge should contain this information.

Similarly, the `penalty_tag` parameter (Default: `penalty`) in the `PenaltyScorer` can be used to represent a static unweighted cost to add to the sum total of the edge scoring plugins.
This is useful to penalize certain routes over others knowing some application-specific information about that route segment. Technically this may be negative to incentivize rather than penalize.
By convention, we say the `penalty` attribute of an edge should contain this information.

## Etc. Notes

---

# Steve's TODO list

- [ ] Create basic file format for graph + parser: OSM and geoJSON

- [ ] feedback operation interface for monitoring





- [ ] Implement live route analyzer: tracking the current edge / node / next node + operation plugin header(s) (file/plugin centric APIs) 
  - Both file and plugin centric (plugin IDs what it cares about, objects report what plugins it cares about)
  - operations have metadata for them, but also have access to edge/node info too

- [ ] operations plugins: call ext server for operation base class + useful demo plugin(s) - realistic, pause at waypoints for signal, node and edge operations, speed limit change operation

- [ ] Implement collision monitor + replanning / rerouting on invalidation for other reasons (requested, time) from up to last_node_passed. Or other than literal invalidations, have regularized replannings based on BT use?

- [ ] other plugins: path manipulator, planners, replanning / rerouting triggerers (collision monitor, frequency, etc)

- [ ] metadata with nested items

- [ ] Quality: BT nodes, Python API, editing and visualization of graphs, testing (plan planner + server + added live stuff), documentation, tutorial (bt change, plugin customize, file field examples)

# Questions

- How can BT trigger BT node again if its still feedback-pending? preemption! Make sure works
- service for closed edge // topic for costmaps come in during request?

Regions polygons vs nodes ??? 
  locus on zones + local planning ehuristic to be within
  for speed zones