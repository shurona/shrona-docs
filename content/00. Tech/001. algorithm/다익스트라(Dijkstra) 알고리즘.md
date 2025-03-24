---
draft: "true"
---
# 최단 경로 탐색 알고리즘

DP 문제인 이유는 최단 거리는 여러 개의 최단 거리로 이루어져 있기 대문이다.

1. 출발 노드를 설정한다.
2. 출발 노드를 기준으로 각 노드의 최소 비용을 저장한다.
3. 방문 하지 않는 노드 중에서 가장 비용이 적은 노드를 선택한다. (이거 중요한 듯)
4. 해당 노드를 거쳐서 특정한 노드로 가는 경우를 고려하여 최소 비용을 갱신한다.

중간 노드가 있을 경우 나눠서 한다 ⇒ 이거 괜찮은 것 같아요

# 구현 정리
## 구현 순서
1. 들어온 값을 노드로 저장
```Java
List<List<Node>> dp = new ArrayList<>();  
for (int i = 0; i <= nodeCt; i++) {  
    dp.add(new ArrayList<>());  
}  
  
for (int i = 0; i < firstLine[1]; i++) {  
    int[] input = Arrays.stream(reader.readLine().split(" "))  
        .mapToInt(s -> Integer.parseInt(s))  
        .toArray();  
  
    dp.get(input[0]).add(new Node(input[1], input[2]));  
    dp.get(input[1]).add(new Node(input[0], input[2]));  
  
}
```

1.  우선순위큐를 사용해서 다음 노드들을 저장한다.
```Java
Queue<Node> queue = new PriorityQueue<>(((o1, o2) -> o1.value - o2.value));  
```

1. 큐가 빌 때까지 돌면서 방문한 노드를 패스하면서 현재 위치에서 최소 거리의 Node로 간다.
```Java
while (!queue.isEmpty()) {  
	Node current = queue.poll();  

	// 현재 노드가 방문한 노드와 같으면 멈춘다.
	if (current.dist == nodeCt) {  
		break;  
	}  

	for (Node node : dp.get(current.dist)) {  
		// 방문한 노드는 다시 가지 않는다.  
		if (visited[node.dist]) {  
			continue;  
		}  

		// 하나씩 순회를 하면서 최소 데이터로 업데이트를 해줘야 한다.  
		// 이전에서 현재 위치 까지 가는 거리 or 시작에서 바로 가는 방향  
		if (answer[node.dist] > answer[current.dist] + node.value) {  
			answer[node.dist] = answer[current.dist] + node.value;  
			queue.add(new Node(node.dist, answer[node.dist]));  
		}  

	}  

}
```

1. 이후 저장된 데이터에서 목적지들의 최소 시간들을 알 수 있다.
```Java
// dp : 특정 Node에서 갈 수 있는 거리의 시간들을 기록하는 배열
dp[nodeCt];
```
## 구현 중 발생한 문제
### 배열의 크기로 인한 메모리 초과
문제에서 입력 노드의 크기가 50000이였고 이를 기록하기 위해서 `50000 x 50000`의 2차원 배열을 선언해서 구현을 하였었으나   
`50000 * 50000 * 4bytes`가 필요하므로 기가 단위의 메모리가 필요해진다.
따라서 이를 해결하기 위해서는 ArrayList를 사용해서 들어온 graph만을 이용해서 데이터를 저장하고 읽는다.
```Java
// 데이터를 기록하기 위한 dp 리스트
List<List<Node>> dp = new ArrayList<>();  
for (int i = 0; i <= nodeCt; i++) {  
    dp.add(new ArrayList<>());  
}

...

// 저장된 node만 확인해준다.
for (Node node : dp.get(current.dist)) {  
	...
}
```
## 발전된 접근
### 중간에 경유지를 거치는 경우
중간에 경유지를 거치면서 최소 거리를 구하는 경우 경유지부터 출발지와 도착지까지의 최소 거리를 구해서 더해주면 된다.

