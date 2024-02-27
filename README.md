A Simple A\* Path-Finding Example in C#
=======================================

A few weeks back, I needed a path-finding solution for a little home project of mine, a game in which a character must move from location to location without walking through walls or other obstacles. A bit of research showed that an algorithm called A\* (pronounced “A Star”) underpins most solutions to this problem—not just in games but also in other applications such as satellite navigation devices.

![Header Image](/Resources/Images/AStarHeader.jpg?raw=true)

There must be thousands of pages on the Internet discussing this topic. The write-up I’ve used most frequently for guidance is Patrick Lester’s article, _[A\* Pathfinding for Beginners](https://web.archive.org/web/20170505034417/http://www.policyalmanac.org/games/aStarTutorial.htm "A* Pathfinding for Beginners")_. I’d recommend that article as a good place to get to grips with the fundamentals. The Wikipedia article, _[A\* search algorithm](https://web.archive.org/web/20170505034417/http://en.wikipedia.org/wiki/A*_search_algorithm "A* search algorithm on Wikipedia")_, is also a helpful resource.

Before I start, take note that this is a really dumb, stripped-back, bare-bones implementation that won’t always give the best result. It’s very much a learning exercise for me, so it’s likely to be wrong in places. What I hope it will do is provide a working example that’s easy to follow and extend. There are much better example implementations around and I’ll include links to a couple of them at the bottom of the page.

First Steps
-----------

I’ll start with a grid of `boolean`s, where `false` means the location is blocked (e.g. a wall) and `true` means it’s clear. Then I’ll identify two points on the grid: a start location and a finish location.

Here’s a representation of my sample grid. It’s 7×5 and includes an L-shaped wall with a one-node-wide gap along the bottom.

![A simple grid with a start and end location](/Resources/Images/AStarInitialMap.png)

_Figure 1: A simple grid with a start and end location_

The above information will be used to initialise the path-finding class. I create a class called SearchParameters to be the container for this information.

    public class SearchParameters
    {
        public Point StartLocation { get; set; }
        public Point EndLocation { get; set; }
        public bool[,] Map { get; set; }
        ...
    }

**Note**: I’m using the `Point` structure from the `System.Drawing` assembly to store grid coordinates.

Internally, the algorithm needs to hold a little more information about each node. It needs to keep a record of a few things as it goes along:

**G**: The length of the path from the start node to this node.  
**H**: The straight-line distance from this node to the end node.  
**F**: An estimate of the total distance if taking this route. It’s calculated simply using F = G + H.

Figure 2 gives a visual representation of how these values are calculated for the node immediately to the right of the start node. The distance along the path so far (G) is 1 step. Using Pythagoras’ theorem, the estimated distance from here to the end node (H) is 3 steps. Adding these two together gives the total estimated ‘cost’ in taking this path (F) of 4 steps.

![Calculating G, H and F](/Resources/Images/AStarFirstExampleStep.png)  
_Figure 2: Calculating F, G and H_

The calculations are repeated for each adjacent node.

![The 'total estimated cost' (F) is calculated for each adjacent node](/Resources/Images/AStarFirstAdjacentNodes.png)  
_Figure 3: The ‘total estimated cost’ (F) is calculated for each adjacent node_

Now that the F-value for each node is known, it can be used to work out which path to try out first by putting all the options in a list and sorting them by F. Clearly, the node at grid location 2,2 with F=4 is the best bet.

![Adjacent nodes sorted by F-value](/Resources/Images/AStarFirstStepList.png)  
_Figure 4: The adjacent nodes are sorted in ascending order of F-value_

This seems like a good place to introduce the `Search(...)` method that’s core to the implementation. The code listing below is incomplete but I’ll build upon it later. It shows the method taking `currentNode` as its starting point, getting a list of any adjacent nodes that are walkable, and then sorting the list by F-value before iterating over its contents.

    private bool Search(Node currentNode)
    {
        ...
        List<Node> nextNodes = GetAdjacentWalkableNodes(currentNode);
        nextNodes.Sort((node1, node2) => node1.F.CompareTo(node2.F));
        foreach (var nextNode in nextNodes)
        {
            ...
        }
        ...
    }

So now the process is repeated using the node at location 2,2. However, there’s some more information that needs to be recorded about each node before moving on much further.

Any nodes that have been added to an ‘adjacent nodes’ list like the one above are marked as ‘Open’, i.e. they’re considered an open option for the search. However, as soon as a node becomes part of a path, it’s marked as ‘Closed’ and it remains closed even if that path ends up being discarded. Marking a node as closed means it won’t be considered again.

![The search moves first to the node with the lowest F-value](/Resources/Images/AStarSecondExampleStep.png)  
_Figure 5: The search moves first to the node with the lowest F-value_

In addition, every ‘Open’ node is given a reference to its ‘Parent’ node so that the path to get there can be traced back to the start. The starting node doesn’t have a parent.

So now the node needs to store the following information:

**G**: The length of the path from the start node to this node.  
**H**: The straight-line distance from this node to the end node.  
**F**: Estimated total distance/cost.  
**Open/closed state**: Can be one of three states: not tested yet; open; closed.  
**Parent node**: The previous node in this path. Always null for the starting node.  
**Is walkable**: Boolean value indicating whether the node can be used.  
**Location**: Keep a record of this node’s location in order to calculate distance to other locations.

In code, it looks something like this:

    public class Node
    {
        public Point Location { get; private set; }
        public bool IsWalkable { get; set; }
        public float G { get; private set; }
        public float H { get; private set; }
        public float F { get { return this.G + this.H; } }
        public NodeState State { get; set; }
        public Node ParentNode { get { ... } set { ... } }
        ...
    }

    public enum NodeState { Untested, Open, Closed }

The next part shows where the ‘Open’ and ‘Closed’ node states comes into play. Just as before, the algorithm needs to calculate the G-, H- and F-values of each adjacent node. This time, though, the ‘Closed’ node at 1,2 is ignored, along with the non-walkable nodes to the right. This leaves four ‘Open’ nodes that have already had their G-, H- and F-values calculated on the basis that the node at 1,2 was their direct parent.

This seems like a reasonable point to show the implementation of `GetAdjacentWalkableNodes(...)`. For each adjacent node, it filters out the ones that are outside the grid’s boundaries, that aren’t walkable, that are ‘Closed’, and that are already on an ‘Open’ list and can’t be reached more efficiently via the current route. The complete listing is as follows:

    private List<Node> GetAdjacentWalkableNodes(Node fromNode)
    {
        List<Node> walkableNodes = new List<Node>();
        IEnumerable<Point> nextLocations = GetAdjacentLocations(fromNode.Location);
    
        foreach (var location in nextLocations)
        {
            int x = location.X;
            int y = location.Y;
    
            // Stay within the grid's boundaries
            if (x < 0 || x >= this.width || y < 0 || y >= this.height)
                continue;
    
            Node node = this.nodes\[x, y\];
            // Ignore non-walkable nodes
            if (!node.IsWalkable)
                continue;
    
            // Ignore already-closed nodes
            if (node.State == NodeState.Closed)
                continue;
    
            // Already-open nodes are only added to the list if their G-value is lower going via this route.
            if (node.State == NodeState.Open)
            {
                float traversalCost = Node.GetTraversalCost(node.Location, node.ParentNode.Location);
                float gTemp = fromNode.G + traversalCost;
                if (gTemp < node.G)
                {
                    node.ParentNode = fromNode;
                    walkableNodes.Add(node);
                }
            }
            else
            {
                // If it's untested, set the parent and flag it as 'Open' for consideration
                node.ParentNode = fromNode;
                node.State = NodeState.Open;
                walkableNodes.Add(node);
            }
        }
    
        return walkableNodes;
    }

Applying the above code to the example scenario, it performs a ‘what-if’ calculation to determine whether it’s more efficient to reach any of the adjacent nodes via location 2,2 than via their existing parent. Note that H doesn’t need to be recalculated.

![The adjacent nodes can be reached more efficiently via a different route](/Resources/Images/AStarSecondExampleStepPart2.png)  
_Figure 6: The adjacent nodes can be reached more efficiently via a different route_

It’s clear from these ‘what-if’ F-values that it’s less efficient to go via this node than via the starting node to reach any of the four locations accessible from 2,2.

**Note**: If it turns out to be more efficient to reach an Open node via the current node, the Open node’s parent is changed to the current node and the ‘what-if’ G- and F- values are applied to it. Unfortunately, this situation isn’t encountered in this walkthrough.

A dead end has been reached, so what now?

Searching Beyond the First Node
-------------------------------

In the sample code, I’ve used a recursive `Search(...)` method. A dead end situation is identified by the absence of any nodes to move to from the current location. To put it another way, the search will only continue along a given path if there’s somewhere to go next. I’ll expand on the earlier code listing for `Search(...)`.

    private bool Search(Node currentNode)
    {
        currentNode.State = NodeState.Closed;
        List<Node> nextNodes = GetAdjacentWalkableNodes(currentNode);
        nextNodes.Sort((node1, node2) => node1.F.CompareTo(node2.F));
        foreach (var nextNode in nextNodes)
        {
            ...
            if (Search(nextNode)) // Note: Recurses back into Search(Node)
                return true;
        }
        return false;
    }

If a dead end is detected, `Search(...)` simply returns `false` so that control is returned to the next level up in the call stack, and that’s exactly what happens here.

Returning to the search from the starting node, the next choice could be either at location 2,1 or at location 2,3 since they both have the same F-value and share #2 position in the list of adjacent nodes. For brevity’s sake, let’s assume the algorithm chooses the node at 2,1 next.

![Try the next-best option](/Resources/Images/AStarThirdExampleStep.png)  
_Figure 7: Try the next-best option_

The Open node at 1,1 is left alone since there’s no advantage going via this route. That leaves three possibilities, with the node at 3,0 achieving the lowest F-value.

![Crossing the corner to location 3,0](/Resources/Images/AStarFourthExampleStep.png)  
_Figure 8: Crossing the corner to location 3,0_

**Note**: This implementation permits the corners of obstacles to be crossed as shown in Figure 8. While it’s arguably valid in this situation, it probably wouldn’t be considered valid if there were an obstacle at location 2,0 (to the lower-left of the blue line). It’s worth considering how to deal with corners and diagonal obstacles if implementing A\*.

From 3,0 there’s only one possible next node: the one at 4,0.

![Crossing to location 4,0](/Resources/Images/AStarFifthExampleStep.png)  
_Figure 9: Crossing to location 4,0_

Of the two next options, crossing the corner to 5,1 is the best.

![Crossing the corner to location 5,1](/Resources/Images/AStarSixthExampleStep.png)  
_Figure 10: Crossing the corner to location 5,1_

There are now five options. Unsurprisingly, the node with the lowest F-value is also the finish node.

![Reaching the finish node](/Resources/Images/AStarSeventhExampleStep1.png)  
_Figure 11: Reaching the finish node_

I’ll expand upon `Search(...)` one last time. The additional code checks whether the last node has been reached and returns `true` if it has. `GetAdjacentWalkableNodes(...)` has already set the parent node on the finish node, so nothing else needs to happen inside this method.

    private bool Search(Node currentNode)
    {
        currentNode.State = NodeState.Closed;
        List<Node> nextNodes = GetAdjacentWalkableNodes(currentNode);
        nextNodes.Sort((node1, node2) => node1.F.CompareTo(node2.F));
        foreach (var nextNode in nextNodes)
        {
            if (nextNode.Location == this.endNode.Location)
            {
                return true;
            }
            else
            {
                if (Search(nextNode)) // Note: Recurses back into Search(Node)
                    return true;
            }
        }
        return false;
    }

Back to the example scenario: the new condition is hit, so the method returns `true`. This brings the application all the way back up the call stack to where `Search(...)` was first called. Returning `true` lets the caller know that a path was found.

Compiling the Path
------------------

Building a list of the nodes that comprise the path is simple: starting at the finish node, follow the line of ancestors, adding the location of each to a new list, until a `null` parent is reached (i.e. the start node has been reached).

![Follow successive parent nodes to build the list of locations along the path](/Resources/Images/AStarFollowParents.png)  
_Figure 12: Follow successive parent nodes to build the list of locations along the path_

The result is a list of locations running _backwards_ from the finish location to the start location. The list is reversed so that they can be returned in the correct order. It can all be wrapped up in a public method along these lines:

    public List<Point> FindPath()
    {
        List<Point> path = new List<Point>();
        bool success = Search(startNode);
        if (success)
        {
            Node node = this.endNode;
            while (node.ParentNode != null)
            {
                path.Add(node.Location);
                node = node.ParentNode;
            }
            path.Reverse();
        }
        return path;
    }

If a path isn’t found, it just returns an empty list, otherwise it returns a list of locations starting with the first adjacent location to the starting point and ending with the finish point.

The sample project includes a console application that shows the algorithm being run across three different grids: one that’s completely open; one with the L-shaped obstacle used in this walkthrough, and; one in which a wall prevents a path from being found.

![Screenshot of the sample application](/Resources/Images/AStarConsoleAppScreenshot.png)  
_Figure 13: Screenshot of the sample application_

Final Notes
-----------

The goal of this blog post is to show the fundamentals of A\* through a really simple C# implementation. As a result of trying to keep the code short and easy to follow, it’s a bit inefficient and it doesn’t produce particularly good routes. Improving its efficiency shouldn’t be too hard: use lazy instantiation when building the `Node` grid; use fields rather than properties (although, of course, this goes against general .NET design guidelines—see MSDN article _[Properties (C# Programming Guide)](http://msdn.microsoft.com/en-us/library/x9fsa0sw.aspx "Properties (C# Programming Guide) on MSDN")_); the other articles linked from this page highlight other optimisations.

As for ways to find _better_ routes, there are plenty of C# examples around that are far better and richer than this one. [CastorTiu](http://www.codeproject.com/script/Membership/View.aspx?mid=240897 "User profile for CastorTiu on CodeProject") has a really nice demo solution on CodeProject, _[A\* algorithm implementation in C#](http://www.codeproject.com/Articles/15307/A-algorithm-implementation-in-C "A* algorithm implementation in C# on CodeProject")_, that animates the search algorithm and allows the user to tweak a few settings.

I also spent a while examining [Woong Gyu La](http://www.codeproject.com/script/Membership/View.aspx?mid=3782460 "User profile for Woong Gyu La on CodeProject")‘s solution on CodeProject, [EpPathFinding.cs- A Fast Path Finding Algorithm (Jump Point Search) in C# (grid-based)](http://www.codeproject.com/Articles/632424/EpPathFinding-cs-A-Fast-Path-Finding-Algorithm-Jum "EpPathFinding.cs- A Fast Path Finding Algorithm (Jump Point Search) in C# (grid-based) on CodeProject"). It has a nice, clear GUI and allows a few settings to be tweaked. I’d recommend taking at look.

_– Mike Clift_

_t: [@mclift](https://twitter.com/mclift)_
