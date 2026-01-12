# Detailed DSA + Boss Room Study Plan

## INSTRUCTIONS
1. Copy the CSV section below into Excel
2. Data → Text to Columns → Delimited → Comma
3. Fill in your start date
4. Each TutorialsPoint article is a separate link you can read
5. Each LeetCode problem has the problem number for easy search

---

## CSV FORMAT

```csv
Day,Week,DSA Topic 1 (TutorialsPoint),DSA Topic 2 (TutorialsPoint),DSA Topic 3 (TutorialsPoint),LeetCode Problem 1,LeetCode Problem 2,LeetCode Problem 3,Boss Room Guide,Boss Room Practice,Done
1,1,DSA - Home,DSA - Overview,DSA - Environment Setup,#1 Two Sum (Easy),#217 Contains Duplicate (Easy),,README.md + A1_code_navigation_guide.md,Import project and play game,
2,1,DSA - Algorithms Basics,DSA - Asymptotic Analysis,,#121 Best Time to Buy and Sell Stock (Easy),#53 Maximum Subarray (Easy),,01_architecture_principles.md,Find SOLID examples in code,
3,1,DSA - Data Structure Basics,DSA - Data Structures and Types,,#704 Binary Search (Easy),#35 Search Insert Position (Easy),,02_clean_code_patterns.md,Identify clean code patterns,
4,1,DSA - Array Data Structure,,,#26 Remove Duplicates from Sorted Array (Easy),#88 Merge Sorted Array (Easy),,18_character_system_deepdive.md,Open ServerCharacter.cs,
5,1,DSA - Array continued,Practice Arrays,,#189 Rotate Array (Medium),#238 Product of Array Except Self (Medium),,18_character_system_deepdive.md,Open ClientCharacter.cs,
6,1,Review Arrays,,,#169 Majority Element (Easy),#283 Move Zeroes (Easy),#448 Find All Numbers Disappeared (Easy),03_networking_essentials.md,Compare Server vs Client,
7,1,REST,REST,REST,REST,REST,REST,REST,REST,
8,2,DSA - Linked List Data Structure,,,#206 Reverse Linked List (Easy),#21 Merge Two Sorted Lists (Easy),,04_design_patterns.md,Find Observer pattern,
9,2,DSA - Doubly Linked List Data Structure,,,#141 Linked List Cycle (Easy),#160 Intersection of Two Linked Lists (Easy),,04_design_patterns.md,Find State pattern,
10,2,DSA - Circular Linked List Data Structure,,,#19 Remove Nth Node From End of List (Medium),#24 Swap Nodes in Pairs (Medium),,09_action_system_deepdive.md,Trace MeleeAction,
11,2,DSA - Skip List Data Structure,,,#234 Palindrome Linked List (Easy),#83 Remove Duplicates from Sorted List (Easy),,09_action_system_deepdive.md,Study ActionConfig,
12,2,DSA - Stack Data Structure,,,#20 Valid Parentheses (Easy),#155 Min Stack (Medium),,26_scriptableobject_architecture.md,Create new ScriptableObject,
13,2,DSA - Expression Parsing,,,#150 Evaluate Reverse Polish Notation (Medium),#71 Simplify Path (Medium),,05_project_structure.md,Explore folder structure,
14,2,REST,REST,REST,REST,REST,REST,REST,REST,
15,3,DSA - Queue Data Structure,,,#232 Implement Queue using Stacks (Easy),#225 Implement Stack using Queues (Easy),,29_unity_clean_code.md,Study Assembly Definitions,
16,3,DSA - Circular Queue Data Structure,,,#622 Design Circular Queue (Medium),#346 Moving Average from Data Stream (Easy),,21_ai_enemy_system.md,Open AIBrain.cs,
17,3,DSA - Priority Queue Data Structure,,,#703 Kth Largest Element in a Stream (Easy),#23 Merge k Sorted Lists (Hard),,21_ai_enemy_system.md,Study AI states,
18,3,DSA - Deque Data Structure,,,#239 Sliding Window Maximum (Hard),#862 Shortest Subarray with Sum at Least K (Hard),,10_connection_state_machine.md,Draw state diagram,
19,3,DSA - Hash Table,DSA - Hashing Data Structure,,#1 Two Sum (redo),#49 Group Anagrams (Medium),,10_connection_state_machine.md,Trace connection flow,
20,3,DSA - Collision In Hashing,,,#347 Top K Frequent Elements (Medium),#146 LRU Cache (Medium),,11_infrastructure_patterns.md,Study PubSub,
21,3,REST,REST,REST,REST,REST,REST,REST,REST,
22,4,DSA - Searching Algorithms,DSA - Linear Search Algorithm,,#704 Binary Search (redo),#278 First Bad Version (Easy),,11_infrastructure_patterns.md,Find PubSub usage,
23,4,DSA - Binary Search Algorithm,,,#33 Search in Rotated Sorted Array (Medium),#153 Find Minimum in Rotated Sorted Array (Medium),,19_game_flow_deepdive.md,Trace boot sequence,
24,4,DSA - Interpolation Search,DSA - Jump Search Algorithm,,#162 Find Peak Element (Medium),#74 Search a 2D Matrix (Medium),,19_game_flow_deepdive.md,Trace scene transitions,
25,4,DSA - Exponential Search,DSA - Fibonacci Search,,#34 Find First and Last Position (Medium),#69 Sqrt(x) (Easy),,20_session_reconnection.md,Study reconnection,
26,4,DSA - Sublist Search,,,#240 Search a 2D Matrix II (Medium),#4 Median of Two Sorted Arrays (Hard),,23_unity_gaming_services.md,Study Relay setup,
27,4,Review Searching,,,#35 Search Insert Position (redo),#374 Guess Number Higher or Lower (Easy),,28_scene_management.md,Study SceneBootstrapper,
28,4,REST,REST,REST,REST,REST,REST,REST,REST,
29,5,DSA - Sorting Algorithms,DSA - Bubble Sort Algorithm,,#912 Sort an Array (Medium),#88 Merge Sorted Array (redo),,27_error_handling_resilience.md,Find try-catch patterns,
30,5,DSA - Insertion Sort Algorithm,DSA - Selection Sort Algorithm,,#147 Insertion Sort List (Medium),#75 Sort Colors (Medium),,27_error_handling_resilience.md,Add error handling,
31,5,DSA - Merge Sort Algorithm,,,#148 Sort List (Medium),#56 Merge Intervals (Medium),,30_debugging_multiplayer.md,Set up ParrelSync,
32,5,DSA - Shell Sort Algorithm,DSA - Heap Sort Algorithm,,#215 Kth Largest Element in an Array (Medium),#973 K Closest Points to Origin (Medium),,30_debugging_multiplayer.md,Add debug logging,
33,5,DSA - Quick Sort Algorithm,,,#912 Sort an Array (Quick Sort),#179 Largest Number (Medium),,22_ui_architecture.md,Study IPUIMediator,
34,5,DSA - Bucket Sort Algorithm,DSA - Counting Sort Algorithm,DSA - Radix Sort Algorithm,#164 Maximum Gap (Hard),#274 H-Index (Medium),,22_ui_architecture.md,Trace UI update,
35,5,REST,REST,REST,REST,REST,REST,REST,REST,
36,6,DSA - Recursion Algorithms,DSA - Tower of Hanoi Using Recursion,,#509 Fibonacci Number (Easy),#70 Climbing Stairs (Easy),,24_animation_vfx_system.md,Study AnimatorTriggeredFX,
37,6,DSA - Fibonacci Series Using Recursion,,,#50 Pow(x n) (Medium),#22 Generate Parentheses (Medium),,25_level_gameplay_objects.md,Study EnemyPortal,
38,6,DSA - Divide and Conquer,DSA - Max-Min Problem,,#53 Maximum Subarray (D&C approach),#169 Majority Element (D&C),,12_system_flow_diagrams.md,Draw your own diagram,
39,6,DSA - Strassen's Matrix Multiplication,DSA - Karatsuba Algorithm,,#973 K Closest Points (D&C),#215 Kth Largest (D&C),,14_antipatterns_guide.md,Find antipattern examples,
40,6,DSA - Tree Data Structure,DSA - Tree Traversal,,#94 Binary Tree Inorder Traversal (Easy),#144 Binary Tree Preorder (Easy),#145 Binary Tree Postorder (Easy),16_performance_patterns.md,Find object pooling,
41,6,DSA - Binary Search Tree,,,#98 Validate Binary Search Tree (Medium),#230 Kth Smallest Element in BST (Medium),,15_testability_debugging.md,Study VContainer,
42,6,REST,REST,REST,REST,REST,REST,REST,REST,
43,7,DSA - AVL Tree,,,#110 Balanced Binary Tree (Easy),#108 Convert Sorted Array to BST (Easy),,17_architecture_decision_framework.md,Apply framework,
44,7,DSA - Red Black Trees,,,#235 Lowest Common Ancestor of BST (Medium),#236 Lowest Common Ancestor of BT (Medium),,Mini-Project,Create new enemy type,
45,7,DSA - B Trees,DSA - B+ Trees,,#102 Binary Tree Level Order Traversal (Medium),#103 Binary Tree Zigzag Level Order (Medium),,Mini-Project,Continue enemy,
46,7,DSA - Splay Trees,,,#199 Binary Tree Right Side View (Medium),#513 Find Bottom Left Tree Value (Medium),,Mini-Project,Test enemy AI,
47,7,DSA - Graph Data Structure,,,#200 Number of Islands (Medium),#133 Clone Graph (Medium),,Mini-Project,Create UI panel,
48,7,DSA - Depth First Traversal,,,#695 Max Area of Island (Medium),#547 Number of Provinces (Medium),,Mini-Project,Continue UI,
49,7,REST,REST,REST,REST,REST,REST,REST,REST,
50,8,DSA - Breadth First Traversal,,,#994 Rotting Oranges (Medium),#127 Word Ladder (Hard),,Mini-Project,Test UI panel,
51,8,DSA - Spanning Tree,DSA - Topological Sorting,,#207 Course Schedule (Medium),#210 Course Schedule II (Medium),,Mock Interview,Practice with friend,
52,8,DSA - Strongly Connected Components,,,#323 Number of Connected Components (Medium),#684 Redundant Connection (Medium),,Portfolio prep,Document learnings,
53,8,DSA - Greedy Algorithms,DSA - Fractional Knapsack Problem,,#55 Jump Game (Medium),#45 Jump Game II (Medium),,Portfolio prep,Update resume,
54,8,DSA - Prim's Minimal Spanning Tree,DSA - Kruskal's Minimal Spanning Tree,,#1584 Min Cost to Connect All Points (Medium),#1135 Connecting Cities With Minimum Cost (Medium),,Review all guides,Write summary,
55,8,DSA - Dijkstra's Shortest Path Algorithm,,,#743 Network Delay Time (Medium),#787 Cheapest Flights Within K Stops (Medium),,Final review,Prepare talking points,
56,8,REST,REST,REST,REST,REST,REST,REST,REST,
57,9,DSA - Dynamic Programming,,,#70 Climbing Stairs (redo with DP),#198 House Robber (Medium),,Interview prep,Mock interview,
58,9,DSA - Matrix Chain Multiplication,,,#322 Coin Change (Medium),#518 Coin Change 2 (Medium),,Interview prep,Review weak topics,
59,9,DSA - Floyd Warshall Algorithm,,,#1143 Longest Common Subsequence (Medium),#72 Edit Distance (Hard),,Interview prep,Final practice,
60,9,DSA - 0-1 Knapsack Problem,,,#416 Partition Equal Subset Sum (Medium),#494 Target Sum (Medium),,Interview prep,Mental preparation,
61,9,DSA - Longest Common Sub-sequence,,,#300 Longest Increasing Subsequence (Medium),#673 Number of LIS (Medium),,REST,Light review,
62,9,Final Review,Final Review,,Mixed practice,Mixed practice,Mixed practice,REST,REST,
63,9,REST,REST,REST,REST,REST,REST,Interview Ready!,Good luck!,
```

---

## TUTORIALSPOINT BASE URL
All articles are at: `https://www.tutorialspoint.com/data_structures_algorithms/`

Example: For "DSA - Array Data Structure", the URL is:
`https://www.tutorialspoint.com/data_structures_algorithms/array_data_structure.htm`

---

## LEETCODE BASE URL
All problems are at: `https://leetcode.com/problems/`

Example: For "#1 Two Sum", the URL is:
`https://leetcode.com/problems/two-sum/`

---

## PROBLEM DIFFICULTY LEGEND
- (Easy) = Good for learning concept
- (Medium) = Interview level
- (Hard) = Stretch goal

---

## WEEKLY FOCUS SUMMARY

| Week | DSA Focus | Key Problems | Boss Room Focus |
|------|-----------|--------------|-----------------|
| 1 | Arrays basics | Two Sum, Maximum Subarray | Architecture intro |
| 2 | Linked Lists, Stacks | Reverse LL, Valid Parentheses | Action system |
| 3 | Queues, Hash Tables | LRU Cache, Top K Frequent | Connection states |
| 4 | Searching algorithms | Binary Search variants | Game flow |
| 5 | Sorting algorithms | Merge/Quick sort problems | Debugging |
| 6 | Recursion, Trees intro | Tree traversals | UI + VFX |
| 7 | Trees advanced, Graphs | Number of Islands | Mini-projects |
| 8 | Graphs, Greedy | Course Schedule, Dijkstra | Interview prep |
| 9 | Dynamic Programming | Coin Change, LCS | Final review |
