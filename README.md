# Dustminer

DustMiner is a two-stage sequential pattern mining framework for **fault detection** in event logs. DustMiner is a diagnostic tool that leverages an extensible framework for uncovering root causes of
failures and performance anomalies in wireless sensor network applications in an automated way.

## Repository Structure
```
DustMiner/
│
├── libraries/
│   ├── anomaly_detection.py
│   └── theft_protection_config.txt
|   └── utils.py
├── dustminer.ipynb   # main file which contain the implementation
├── dustminer_event_length.ipynb
├── README.md
└── requirements.txt
```
1. In libraries folder, we have utils.py file which contains helper functions like get_paths(), read_traces() which are used to make main implementation easier and to make code look clean.
2. dustminer.ipynb - This jupyter notebook contains the main implementation of the Dustminer. For all the functions implemented, we have added the description there itself.
3. dustminer_event_length.ipynb - This notebook almost looks like the dustminer.ipynb file but only change here is how we use the data to dustminer. Here we use different lengths to get more insights about the result.
4. requirements.txt - This file contains all the required libraries to be installed in the system inorder to successfully use the implementation.

### How to use the repository
1. Install the required libraries using the command below.
```
pip install -r requirements.txt
```
2. Open the dustminer.ipynb file and ensure the data path are correct as per the implementation. If not then adjust accordingly.
3. Here normalbase_path indicates normal train data path and faultybase_path indicates the test data path.
4. The data we have is in the format [id,timestamp] as tuple so we use load_data() function to correctly pull out 'id' alone from tuple. Adjust the function accordingly if your data is not tuple.
5. Set the values for MIN_SUP, SEGMENT_WIDTH, Normalize_support. In our scenario we used MIN_SUP=2, SEGMENT_WIDTH=50 and normalize support = True.
6. Next we create a new file which is the ground truth file. Here we create in the below format.
```
{
    "trace_trial1_1090-1140.json": [[12,6,7,8,9,6]],
    "trace_trial1_1175-1225.json": [[9,6,7,8,6,7,8]],
}
```
The ground truth file we had was for each trace_trial we had all the indices were the actual anomaly existed. So we need to extract them and create the new ground truth like below with each filename and then the actual anomaly sequence.  
We have used get_ground_truth_file(), get_trace_info(), find_sequence_ground_truth(), create_labels() function to perform the same.
8. Using mine_discriminative_patterns_progressive(), we compute the discriminative pattern which is the stage1. Here we need to pass good_seq, bad_seq, min_sup, theta(0.8), delta(0.8), k_start=1,k_max=length of S1 cartesian, normalize, repeated_event_id.
```
good_sequence = [[6,7,8,9,6,7,8,9,10,11,12],[6,7,8,9,10,11,12,13,6,7,8,6,7,8,9]]
bad_sequence = [[6,7,8,9,6,7,8,7,9,10],[6,7,8,9,10,11,12]]
discriminative_patterns = mine_discriminative_patterns_progressive(good_sequences,bad_sequences,min_sup=7,theta=0.8, delta=0.8, k_start=1,k_max=k_max_1,stop_when_found=False,normalize=True,repeated_event_id=False)
```
These are the values we have used for theta, delta.
9. Now when we print the discriminative patterns, we could see like below.
```
(23, 24, 49, 56, 57, 65, 70, 71): {'bad': 0.0167548031686215,'good': 0.010874626452331941,'delta': 0.00588017671628956},
```
10. Once discriminative patterns are ready, we proceed with stage2 mining. Here the main idea is again mining only in the area of discriminative patterns inorder to get more clarity in the root cause pattern.
11. Using the function stage2_patterns(), we perform stage2 mining.
```
stage2_patterns = stage2_mining(bad_seqs=bad_sequences,discriminative_patterns=discriminative_patterns,seg_width=50,top_k=5,max_len_stage2=None,overlap=False)
```
Here we need to pass bad_sequence, discriminative_patterns, seg_width=50(default), top_k=5.
```
Pattern	                                                Support
0	(53, 54, 55, 56, 57, 58, 59, 20, 21, 22, 49)	         66
1	(60, 61, 49, 57, 62)	                                 66
```
13. Once stage2 patterns are completed. Then we proceed in calculating the precision, recall and F1 score by comparing the dustminer results with ground truth we already know.
14. Now we iterate the ground truth with filename as key and check if the sequence has a perfect match with any dustminer result sequences or having atleast 60% match with continous match and it should be nonoverlapping then it is marked as 'True positive'.
15. Here when have a True positive, then rest all the sequences are marked as False positive. Also, when no match at all for a test file then we have a False negative count and rest of sequences are marked as False positive.
16. Then we calculate classwise detections for going into more analysis.
17. Also, the code in the last cell is used to identify the errors we detected are localized one's or not. We take the count to observe among the detected ones how many are with exactly one occurence. This gives more insights about the dustminer results.
   
## **Implementation & Alogrithms**
#### **First stage is Discriminative pattern mining**
1. At first, the tool will use a data collection frontend and collect runtime events for post mortem analysis.
2. Once the logs of runtime events are available, the tool will separate the collected sequence into two piles - ‘a good pile’ which contains the part of the log when the system performs as expected
   and a ‘bad pile’ which contains the parts of logs when the system fails or exhibits poor performance.
3. This data separation phase is done based on a predicate that defines ‘good’ versus ‘bad’ behavior which is to be provided by the application developer. Here in our scenario we have considered the train data as 'Good pile' and test data as 'bad pile'.
4. A discriminative frequent pattern mining algorithm then looks for patterns(sequence of events) that exist with very different frequencies in the two piles.
5. Once such a pattern is found, the algorithm traces back to earlier events in the logs that might correlate with the start of the anomaly.
6. Even if those earlier events are rare, they can still be important like early indicators or root causes.

#### **Normal Apriori Algorithm**
The main idea of this algorithm is that if a pattern is frequent, then all its sub-patterns must also be frequent.
1. At the first iteration, it counts the number of occurrences called support of each distinct event in the data set(i.e in the good or bad pile).
2. Next, it will discard all events that are infrequent (their support is less than some parameter minSup).
3. The remaining events are frequent patterns of length 1.
4. Assume the set of frequent patterns of length 1 is S1. And in the next iteration, the algorithm generates all the candidate patterns of length 2 which is S1 * S1.
5. Here ‘*’ represents cartesian products. It will then compute the frequency of occurrence of each pattern in S1 * S1 and then discards those with support less than minSup again.
6. The remaining patterns are the frequent patterns of length 2. Let it be S2.
7. Similarly the algorithm will generate all the candidate patterns of length 3 which is S2 * S1 and discard infrequent patterns(with support less than minSup) to generate S3 and it goes on.
8. This is continued until it cannot generate any more frequent patterns.
9. But here this is some problem so we are extending this algorithm.

#### Example:
Imagine we have 5 sensor event logs
```
1: A,B,C
2: A,c
3: A,D
4: B,C
5: A,B,C,D
```
And we set minSup = 2.
A - 4, B-3, C-4, D-2
S1 = {A, B, C, D} because all have value greater than minSup

Step 2: Here the pairs can be A-B, A-C, A-D, B-C, B-D, C-D
Now we should count their support.
A-B – 2, A-C – 3, A-D – 2, B-C – 3, B-D – 1, C-D – 1
We keep only those with support >=2.
So S2 = {A-B, A-C, A-D, B-C}

Step 3: Here we take triplets from S2.
Here the pairs are A-B-C, A-B-D, A-C-D, B-C-D.
A-B-C – 2
A-B-D – 1
A-C-D – 1
B-C-D – 1
Only A-B-C is frequent. So S3 = {A,B,C}

Step 4: Here we take 4 items.
We take S3 * S1, so only one possible A-B-C-D.
Here the support value of this pattern is 1 and it gets discarded because it does not meet minSup value.
So finally the frequent patterns with minSup=2 are
```
Length 1: A,B,C,D
Length 2: A-B, A-C, A-D, B-C
Length 3: A-B-C
```

#### **Drawbacks of the Normal apriori and the solutions we have introduced to overcome the challenges**
1. False frequent patterns in apriori
2. Suppressing redundant sub sequences
3. Two stage mining for infrequent events
4.  Inconsistent event volume between runs
5. Event parameters cause alphabet explosion

### Modified Apriori Algorithm
1. Divide dataset into two piles — a good pile (normal traces) and a bad pile (faulty traces)
2. Construct dynamic windows for every sequence: A window spans from one occurrence of an event ID to its next repetition in the same trace.
   Example: [6,7,8,6,7,8,9,10] → windows: [6,7,8], [6,7,8,9,10]
3. Perform frequent pattern mining inside dynamic windows instead of over full sequences, ensuring support counts reflect localized repeated behaviour.
4. Mine frequent continuous patterns progressively for pattern lengths K = 1 … k_max(value of k_max is the length of S1 cartesian). Generate length-K candidates by right-extending length-(K–1) patterns.
5. Keep only candidates whose window-support ≥ min_sup.
6. Compute support statistics separately for good and bad piles.
   1. Across-sequence presence: how many sequences contain the pattern at least once.
   2. Average in-sequence frequency: how often the pattern occurs in sequences where it is present.
7. Detect discriminative patterns at each K using ratio tests.
   1. A pattern is discriminative for the bad pile if it appears significantly more often in bad traces than in good traces, using thresholds θ (Across-sequence presence ratio threshold) and δ (In-sequence average count ratio threshold).
   2. Here we are using the value as 0.8 for both delta and theta.
8. Collect all discriminative patterns found across lengths for the bad pile.
9. The iteration is done till k_max or when the set in the current cartesian does not satisfy the min_sup criteria. In both of these scenario, we stop the iteration.
10. At each stage we compress the patterns if the pattern is a subsequence and has the same support.
11. For each discriminative patterns, calculating its probability of occurrences in bad vs good sequences using conditional probabilities.
12. Score patterns using d = P_bad - P_good and we keep only patterns with positive value of d and then ranking the patterns in descending order.
13. Here while taking the cartesian products we are considering both scenario's to check how it performs. We use the flag 'repeated_event_id' while calling the mine_discriminative_patterns_progressive function. If the flag value is true then we allow repeated event_id
    while forming the cartesian and if the flag value is false, then we restrict the same event_id to come in the cartesian. 

### Stage 2 Mining
1. After we get the discriminative patterns list, we again perform the mining particularly in the areas of the discriminative pattern inorder to get the exact root cause area.
2. We take bad sequences and discriminative patterns as input for Stage 2 mining.
3. We split each faulty sequence into segements. The segment width we have used is 50.
4. For every segment, we need to count how many discriminative patterns from Stage 1 appear inside it. This gives each segment a score which tells which is the most likely faulty segment.
5. Merging the consecutive segments that are directly adjacent and both have positive scores. This gives longer continuous faulty regions.
6. Rank all the merged regions by their score and their length.
7. Once ranking is completed, we need to mine patterns inside each selected region. For each of these regions, perform frequent continuous pattern mining within that region alone.
8. Similar to Stage 1, we can remove the redundant shorter patterns that are fully covered by longer ones with same support.
9. Keep only those patterns that appear in all of the selected top faulty regions. Aggregrate their supports over region.
10. Now we get the final pattern list which is the most likely root cause patterns.


