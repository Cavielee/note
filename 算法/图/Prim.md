# Prim 算法

Prim 算法，查找最小生成树。

找出能够连连通所有节点的边组合，并要求该组合weight和最小

**约束：**无向图



**思路：**

1. 准备一个nodeSet存储当前处理过的节点，一个priorityQueue存储当前所有可选的边（按照weight排序）
2. 从一个节点出发，把该节点加入到nodeSet，并将其所有关联的边加入priorityQueue
3. 从priorityQueue中依次选出最小边，每次进行如下判断
   * 如果边的to节点未加入过nodeSet，则将该节点加入nodeSet，并将其关联的边加入priorityQueue
   * 如果边的to节点加入过nodeSet，则忽略



> Prim 算法和 Kruskal算法区别：
>
> 1. 由于Prim 算法是每一加入一个Node后，才加入新的可选的边，因此可能出现重复判断相同的边。而 Kruskal 算法则是一开是对所有的边进行权重排序，然后一次判断每一条边是否能加入，效率上 Kruskal 更优。
> 2. Kruskal 算法需要一开始知道所有的边，而 Prim 算法只需要从一个节点就可以直接搜索。

```java
/**
 *  Prim 算法，查找最小生成树
 *  约束：无向图
 *  找出能够连连通所有节点的边组合，并要求该组合weight和最小
 *  思路：
 *  1. 准备一个nodeSet存储当前处理过的节点，一个priorityQueue存储当前所有可选的边（按照weight排序）
 *  2. 从一个节点出发，把该节点加入到nodeSet，并将其所有关联的边加入priorityQueue
 *  3. 从priorityQueue中依次选出最小边，每次进行如下判断
 *        如果边的to节点未加入过nodeSet，则将该节点加入nodeSet，并将其关联的边加入priorityQueue
 *        如果边的to节点加入过nodeSet，则忽略
 * @author CavieLee
 * @since 2022/04/11
 */
public class Prim {
    public static Set<Edge> primMST(Graph graph) {
        // 对所有边排序
        PriorityQueue<Edge> priorityQueue = new PriorityQueue<>((o1, o2) -> {
            return o1.weigh - o2.weigh;
        });
        Set<GraphNode> nodeSet = new HashSet<>();
        Set<Edge> result = new HashSet<>();

        for (GraphNode node : graph.nodes.values()) {
            if (!nodeSet.contains(node)) {
                // 以该节点出发
                nodeSet.add(node);
                // 先添加该节点的关联的边
                priorityQueue.addAll(node.edges);
                while(!priorityQueue.isEmpty()) {
                    Edge cur = priorityQueue.poll();
                    GraphNode toNode = cur.to;
                    // 判断nodeSet是否包含该to节点
                    if (!nodeSet.contains(toNode)) {
                        nodeSet.add(toNode);
                        result.add(cur);
                        priorityQueue.addAll(toNode.edges);
                    }
                }
            }
//            break;
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
        Set<Edge> edges = primMST(graph);
        edges.forEach(edge -> {
            System.out.println(edge.weigh);
        });
    }
}

```

