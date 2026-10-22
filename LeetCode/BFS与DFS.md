# BFS

BFS**是一种由近及远的遍历方式，从某个节点出发，始终优先访问距离最近的顶点，并一层层向外扩张**

也就是说 先访问一个节点距离最近的节点 然后层层向外扩张

根据这个性质 我们可以得出BFS实际上是一种**FIFO**的策略（遍历过后就视作已访问） 那么根据FIFO 自然想到要用`queue`来解决

同时 由层层扩张 我们可以知道这里采取邻接表的图是最优的

```cpp
// 使用邻接表表示图
class Graph {
    int V; // 顶点数量
    vector<vector<int>> adj; // 邻接表

public:
    Graph(int V) {
        this->V = V;
        adj.resize(V);
    }

    // 添加无向边
    void addEdge(int u, int v) {
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    // BFS 遍历 (基于队列 Queue)
    void BFS(int startNode) {
        vector<bool> visited(V, false);
        queue<int> q;

        // 访问起始节点并入队
        visited[startNode] = true;
        q.push(startNode);

        cout << "BFS 遍历序列: ";
        while (!q.empty()) {
            int curr = q.front();
            q.pop();
            cout << curr << " ";

            // 遍历当前节点的所有邻居
            for (int neighbor : adj[curr]) {
                if (!visited[neighbor]) {
                    visited[neighbor] = true;
                    q.push(neighbor);
                }
            }
        }
        cout << endl;
    }
```

时间复杂度O(|V|+|E|)

# DFS

DFS是**一种优先走到底、无路可走再回头的遍历方式**。

那么不难看出来 这种属于FILO/LIFO

那么和这个匹配的就是`stack`

这个和树的前序、中序、后序遍历很像 而且这三个也属于DFS

```cpp
//DFS 遍历 (基于递归/栈)
    void DFS(int startNode) {
        vector<bool> visited(V, false);
        cout << "DFS 遍历序列: ";
        DFSUtil(startNode, visited);
        cout << endl;
    }

private:
    // DFS 递归辅助函数
    void DFSUtil(int node, vector<bool>& visited) {
        visited[node] = true;
        cout << node << " ";

        // 递归访问所有未访问的邻居
        for (int neighbor : adj[node]) {
            if (!visited[neighbor]) {
                DFSUtil(neighbor, visited);
            }
        }
    }
};
```


或者显示模拟栈

```cpp
//  DFS 遍历 (基于显式模拟栈 std::stack)
void DFS(int startNode) {
    // 1. 初始化访问标记数组
    vector<bool> visited(V, false);
    
    // 2. 创建一个显式栈
    stack<int> s;

    // 3. 将起始节点压入栈中
    s.push(startNode);

    cout << "DFS (模拟栈) 遍历序列: ";

    while (!s.empty()) {
        // 4. 弹出当前栈顶节点
        int curr = s.top();
        s.pop();

        // 5. 如果该节点还未被访问，则进行访问处理
        if (!visited[curr]) {
            visited[curr] = true;
            cout << curr << " ";
        }

        // 6. 将当前节点所有未访问的邻居压入栈中
        // 注意：为了让遍历顺序和递归版完全一致（从小到大），我们可以逆序将邻居压栈
        // 因为栈是后进先出 (LIFO)，后压入的会先被弹出访问
        for (auto it = adj[curr].rbegin(); it != adj[curr].rend(); ++it) {
            int neighbor = *it;
            if (!visited[neighbor]) {
                s.push(neighbor);
            }
        }
    }
    cout << endl;
}

```

一般推荐使用函数调用 除非数据量特别大会导致栈溢出


