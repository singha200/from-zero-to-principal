# DSA prep for Staff Engineer Career Track

- Focus on [grind 75](https://www.techinterviewhandbook.org/grind75/)

# Leetcode problems that make total sense to Devops and SRE Interviews

1. Arrays & Hashing (Most common — log parsing, frequency, deduplication)

- [Two Sum (1)](https://leetcode.com/problems/two-sum/description/)
- [Contains Duplicate (217)](https://leetcode.com/problems/contains-duplicate/)
- [Valid Anagram (242)](https://leetcode.com/problems/valid-anagram/) → string/log pattern matching
- [Group Anagrams (49)](https://leetcode.com/problems/group-anagrams/) → grouping similar log patterns/errors
- [Top K Frequent Elements (347)](https://leetcode.com/problems/top-k-frequent-elements/) - most frequent errors/IPs/status codes in logs
- [Valid Sudoku (36) or similar grid validation](https://leetcode.com/problems/valid-sudoku/) → config validation
- [Longest Consecutive Sequence (128)](https://leetcode.com/problems/longest-consecutive-sequence/) → finding gaps in timestamps/sequence numbers

2. Sliding Window / Two Pointers (Very high relevance — time-series, logs over windows)

- [Longest Substring Without Repeating Characters (3)](https://leetcode.com/problems/longest-substring-without-repeating-characters/) → unique sessions/connections in time window
- [Maximum Average Subarray I (643)](https://leetcode.com/problems/maximum-average-subarray-i/) → average latency/error rate over window
- [Sliding Window Maximum (239)](https://leetcode.com/problems/sliding-window-maximum/) → peak usage/error spikes in metrics
- [Minimum Window Substring (76)](https://leetcode.com/problems/minimum-window-substring/) → smallest log segment containing all error types (rare but asked)
- [Find All Anagrams in a String (438)](https://leetcode.com/problems/find-all-anagrams-in-a-string/) → pattern search in streams/logs

3. Stack / Queue (Parsing nested structures, backpressure, monotonic)

- [Valid Parentheses (20)](https://leetcode.com/problems/valid-parentheses/) → config/JSON/YAML bracket validation
- [Min Stack (155)](https://leetcode.com/problems/min-stack/) → tracking min latency/resource
- [Daily Temperatures (739)](https://leetcode.com/problems/daily-temperatures/) → next greater element (e.g., next scaling event)
- [Number of Recent Calls (933)](https://leetcode.com/problems/number-of-recent-calls/) → recent pings/requests in time window (rate limiting)
- [Implement Queue using Stacks (232)](https://leetcode.com/problems/implement-queue-using-stacks/) → FIFO queue with stack or vice versa

4. Binary Search (Common for thresholds, optimization)

- [Binary Search (704)](https://leetcode.com/problems/binary-search/) → find 
- [Search in Rotated Sorted Array (33)](https://leetcode.com/problems/search-in-rotated-sorted-array/) → find in rotated array
- [Find Minimum in Rotated Sorted Array (153)](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/) → find min in rotated array
- [Time Based Key-Value Store (981)](https://leetcode.com/problems/time-based-key-value-store/) → versioned configs/metrics
- [Capacity To Ship Packages Within D Days (1011)](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/) → bin packing/resource allocation

5. Design / Data Structure Simulation (Hit counter, LRU, rate limiter basics)

- [Design Hit Counter (362)](https://www.geeksforgeeks.org/system-design/design-a-hit-counter/) — extremely common (request counting over sliding time)
- [LRU Cache (146)](https://leetcode.com/problems/lru-cache/) → caching layers, eviction policies in infra tools
- [Logger Rate Limiter (359)](https://leetcode.com/problems/logger-rate-limiter/) → rate limiting logs/alerts
- [Design Twitter (355)](https://leetcode.com/problems/design-twitter/) → social graph (news feed → event ordering)

6. Graphs / Trees (lighter — dependency graphs, service maps)

- [Number of Islands (200)](https://www.geeksforgeeks.org/dsa/find-the-number-of-islands-using-dfs/) → dependency resolution (package/K8s CRD ordering)
- [Course Schedule (207)](https://leetcode.com/problems/course-schedule/) → dependency resolution
- [Clone Graph (133)](https://leetcode.com/problems/clone-graph/) → deep copy of resource graphs
- [Course Schedule II (210)](https://leetcode.com/problems/course-schedule-ii/) → topological sort for rollout order
- [Pacific Atlantic Water Flow (417)](https://leetcode.com/problems/pacific-atlantic-water-flow/) - similar traversal

7. Other occasional high-relevance
- [Longest Increasing Subsequence (300) variants](https://leetcode.com/problems/longest-increasing-subsequence/) → performance trends
- [Merge Intervals (56)](https://leetcode.com/problems/merge-intervals/) → merging time ranges/alert windows
- [Task Scheduler (621)](https://leetcode.com/problems/task-scheduler/) → scheduling with cooldown (cron-like or batch jobs)

- After solving, think: How could this pattern help parse CloudTrail logs / aggregate metrics / detect anomalies / rate-limit API calls?
- Solve in Python — focus on clean code + verbal explanation of time/space + "how this applies to logs/metrics/K8s".
- Practice variations: "Adapt this to find top 5 error types in a streaming log" or "Add sliding window eviction".
- Spend the rest of your prep on system design, platform engineering scenarios, leadership stories, and practical scripting demos.
