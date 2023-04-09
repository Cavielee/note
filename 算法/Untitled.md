# 定义

图指的是节点之间通过边串联起来。

有向图：节点连接的边是一个节点指向另一个节点。

无向图：是一种特殊的有向图，指的是所有边都是两个节点互相指向。

```java
/**
 * 图工具
 * @author CavieLee
 * @since 2022/04/06
 */
public class GraphUtils {

    /**
     * matrix 为二维矩阵，每一行都有三个元素（代表图中的一条边）: {from, to, weight}
     */
    public static Graph createGraph(int [][] matrix) {
        Graph graph = new Graph();
        for (int[] edgeInfo : matrix) {
            int from = edgeInfo[0];
            int to = edgeInfo[1];
            int weight = edgeInfo[2];
            if (!graph.nodes.containsKey(from)) {
                graph.nodes.put(from, new GraphNode(from));
            }
            if (!graph.nodes.containsKey(to)) {
                graph.nodes.put(to, new GraphNode(to));
            }
            GraphNode fromNode = graph.nodes.get(from);
            GraphNode toNode = graph.nodes.get(to);
            Edge edge = new Edge(weight, from, to);
            fromNode.nexts.add(toNode);
            fromNode.out++;
            toNode.in++;
            fromNode.edges.add(edge);
            graph.edges.add(edge);
        }
        return graph;
    }
}
```

