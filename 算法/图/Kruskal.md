# K

Kruskal 算法，查找最小生成树。

找出能够连连通所有节点的边组合，并要求该组合weight和最小

**约束：**无向图



**思路：**

1. 一开始所有节点对应一个只包含自身节点的集合；

2. 对所有边按照weight进行排序；

3. 依次尝试的添加边（依据from节点和to节点的集合是否为同一个进行判断）

   * 如果不是，则添加该边，并将from节点集合和to节点集合合并；

   * 如果是，则忽略该边；

```java
/**
 * Kruskal 算法，查找最小生成树
 * 约束：无向图
 * 找出能够连连通所有节点的边组合，并要求该组合weight和最小
 * 思路：
 * 1. 一开始所有节点对应一个只包含自身节点的集合
 * 2. 对所有边按照weight进行排序
 * 3. 依次尝试的添加边（依据from节点和to节点的集合是否为同一个进行判断）
 *      如果不是，则添加该边，并将from节点集合和to节点集合合并
 *      如果是，则忽略该边
 * @author CavieLee
 * @since 2022/04/11
 */
public class Kruskal {
    // 简易版
    public static class MySet {
        private Map<GraphNode, List<GraphNode>> setMap;
        public MySet(Collection<GraphNode> nodes) {
            setMap = new HashMap<>();
            for (GraphNode node : nodes) {
                setMap.put(node, Arrays.asList(node));
            }
        }

        // 判断两个节点是否在同一个集合
        public boolean isSameSet(GraphNode from, GraphNode to) {
            return setMap.get(from) == setMap.get(to);
        }

        // 合并两个节点的集合
        public void union(GraphNode from, GraphNode to) {
            List<GraphNode> fromSet = setMap.get(from);
            List<GraphNode> toSet = setMap.get(to);

            for (GraphNode node : toSet) {
                fromSet.add(node);
                setMap.put(node, fromSet);
            }
        }
    }

    public static Set<Edge> kruskalMST(Graph graph) {
        MySet mySet = new MySet(graph.nodes.values());
        // 对所有边排序
        PriorityQueue<Edge> priorityQueue = new PriorityQueue<>((o1, o2) -> {
            return o1.weigh - o2.weigh;
        });
        priorityQueue.addAll(graph.edges);

        Set<Edge> result = new HashSet<>();
        while(!priorityQueue.isEmpty()) {
            Edge cur = priorityQueue.poll();
            if (!mySet.isSameSet(cur.from, cur.to)) {
                mySet.union(cur.from, cur.to);
                result.add(cur);
            }
        }
        return result;
    }

    public static void main(String[] args) {
        int[][] matrix = {  {1, 2, 1},
                            {1, 3, 2},
                            {2, 4, 2},
                            {3, 4, 1},
                            {1, 4, 3},
                            {2, 3, 1},
                            };
        Graph graph = GraphUtils.createGraph(matrix);
        Set<Edge> edges = kruskalMST(graph);
        edges.forEach(edge -> {
            System.out.println(edge.weigh);
        });
    }
}
```

