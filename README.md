# Sparse Mixture of Experts
How I accidentally built a Sparse MoE before I knew what it was  

# Part 1

## Backstory

### Boids
It all started with boids. Early in my game development career, I kept encountering the term "spatial partitioning." While researching game design patterns, I discovered Robert Nystrom's Game Programming Patterns. A concise, web based book with a section on performance optimizations. That section covered spatial partitioning alongside topics like data locality and object pooling. I'll return to those later, but for now, my focus remains on spatial partitioning.

I must have googled the term a dozen times, reading everything I could find. Yet I lacked the confidence to attempt an implementation myself. Ironically, I could probably have explained spatial hashes and quadtrees in detail, but I viewed myself as a novice and assumed implementation was beyond my reach. Having never seen working code didn't help. It was difficult to translate theory into practice.

That changed when I started exploring boids: a flocking simulation algorithm created by Craig Reynolds. The concept is elegant. To create realistic looking flocks, each bird compares itself to others in its flock. Birds initially fly in random directions, but three simple rules ensure emergent flocking behavior: separation (avoid crowding neighbors), alignment (steer toward the average heading of neighbors), and cohesion (move toward the average position of neighbors). While individual birds influence their own direction, the algorithm ensures each decision is shaped by both nearby birds and the flock as a whole.

In terms of performance, however, this algorithm is expensive. No offense to Reynolds, inventing the algorithm was achievement enough. In Big O notation, naive boids runs at O(n²) complexity per update. As flock size grows, so does the computational burden. Ten birds require 100 pairwise comparisons; 100 birds require 10,000; and 10,000 birds would require 100 million calculations per frame.

### Spatial Partitioning
While learning about boids, spatial partitioning resurfaced. No matter how much I searched, I struggled to find concrete implementations. I spent significant time coming up empty until one day I found a partially completed spatial hash written in Java on some obscure forum. The key lookup table implementation was remarkably simple, though it used math I didn't immediately understand. I decided to work through a concrete example to figure out how it might work.

First, I drew a grid on paper in the positive integer quadrant, labeling coordinates and cell IDs. Cell ID 1 occupied the upper left corner; the first row contained cells 1–3, the second row 4–6, and the third row 7–9. Then I listed random 2D vectors that could fall within this grid. To map vectors to cell IDs, I needed to determine which row and column each point belonged to. The x coordinate determined the column; the y coordinate determined the row. Translating this to IDs was straightforward once I factored in cell size for scaling. The exact formula escapes me now, but it was simple. Different from the example I'd found, yet it worked in practice.

I already had a boids simulation running, so I modified the nearest neighbors portion to use insertions and lookups against the spatial hash. I also stopped updating every frame, instead updating every n frames. Suddenly, 1,000 boids became instantly possible.

Before continuing, I should mention something I once heard: a golden rule of programming is that any O(n²) operation can often be reduced to O(n) by using a hash map instead of nested array iterations. This made no sense to me at the time. But while implementing the spatial hash, it clicked. The dictionary's O(1) key value lookups meant retrieving nearby neighbors could become effectively O(1) if I used approximate nearest neighbors rather than exhaustive K nearest neighbors. In practice, most agree the full implementation becomes O(n log n): n insertions per frame, with a reduced search space for neighbor queries.

### Spatial Hash Refinement
While my original key ID formula worked, I eventually rewrote it. I'd designed it for positive integer space only, and adapting it for negative coordinates proved frustrating. My solution was to turn the key ID into a vector. Since x and y coordinates already determined the lookup, why not simply "snap" coordinates to a grid cell?

The approach: divide each coordinate by cell size, then truncate to integers. With a cell size of 10, the point (50, −30) maps to cell (5, −3). The point (55.5, −33.3) also maps to (5, −3) after division and truncation. This became my go to method for spatial hashing.

### Collision Detection
Around this time, I was contracting for Color Surge. Coincidentally, I was tackling a major problem: lag on low end mobile devices. I'd made several attempts, each yielding incremental improvements, but never eliminating lag entirely.

One late night, while profiling performance, I noticed something I'd overlooked: collision detection was spiking CPU usage at specific moments. Specifically when the player passed groups of circles. Each circle used polygon colliders to match its visual shape precisely. Every circle had four color segments, meaning four polygon colliders, each defined by 20 vertices.

For levels with many circles, this became problematic. Consider a complex obstacle: circles nested inside circles, stacked in a row. Three nested circles times four colliders equals 12 polygon colliders. Four such stacks means 48 colliders. At 20 vertices each, that's 960 vertices participating in realtime collision detection.

My solution: spatial partitioning. I inserted all obstacles into a spatial hash, identified the nearest obstacle to the player, then inserted that obstacle's collider coordinates into a second spatial hash. A KNN query returned the relevant colliders, which I enabled; all others remained disabled.

The performance gains were dramatic. Low end devices went from unplayable to smooth. On a mid range PC, my stress test scene previously capped at 30 complex obstacles before becoming unplayable. After integrating spatial hashing, hundreds of obstacles ran without performance loss.

### Procedural Content Generation
Another project emerged: procedurally generating levels for Color Surge. The number of times I returned to the drawing board was staggering, though the final solution seems straightforward in retrospect.

Levels in Color Surge are categorized as easy, medium, or hard, each containing a specific number of obstacles. Each obstacle has unique configurations: colors, colliders, names, and more. A level designer manually determined which obstacles appeared in each level.

### Obstacle Encoding
Automating this required analytical thinking. To use obstacles in an algorithm, I needed to encode them into a computable format. Rather than referencing obstacles by name, I assigned each a numeric ID (1–3000), then cast that ID to a character. Each obstacle now had a single character representation that was both human readable and computationally useful.

### Level Encoding
Levels became concatenations of their obstacle encodings. Esentially sequences of characters representing each level's composition.

### Similarity Scoring
The character encoding enabled similarity scoring. Generated levels needed to be unique, not identical or even closely resembling existing levels. I implemented three string similarity algorithms: number of common characters, longest common substring, and longest common subsequence. Rather than choosing one, I combined all three in a weighted average. Each level's representation was compared against all others, producing a similarity percentage. The most similar level (1 nearest neighbor) and its similarity score were always available for each level.

### Difficulty Scoring
Levels could now be generated by randomly selecting obstacles until a low similarity candidate emerged. But this made difficulty random. Which was problematic since levels were grouped by difficulty.

I wanted to assign difficulty scores to obstacles, then sum them per level. But how? The project stalled.

### Neural Networks
A friend once told me neural networks are just "graphs" and asked if I understood graph theory. I knew the basics. With that mental model, I researched further and discovered the multilayer perceptron. I started with single layer perceptrons ie the building blocks of feedforward networks. The underlying math was essentially y = mx + b. The same equation everyone jokes about never needing turns out to be everywhere.

Training via gradient descent was beyond me then, so I turned to genetic algorithms. The concept: randomly initialize weight arrays, cross splice them, evaluate fitness, keep the best performers, discard the rest. Eventually, this yields weights that minimize error. In practice, batches of data pass through perceptrons with spliced weight arrays, and those minimizing error survive to the next generation.

### Multivariate Linear Regression
This framing made the connection to multivariate linear regression obvious: y = w₁x₁ + w₂x₂ + w₃x₃ + b. I wondered if I could apply this to the procedural content generator.

### Feature Vectors
I viewed each obstacle's configuration as a data point. Colors, colliders, names, etc these became feature vectors: multidimensional representations where each dimension captured the presence or count of an attribute. I min-max scaled these vectors to normalize magnitudes. Neural networks consume feature vectors, and now each obstacle had one.

With feature vectors in hand, I considered multivariate linear regression for difficulty scoring. A weight vector of equal length could multiply against the normalized feature vector, with a bias term added essentially a perceptron.

### Reinforcement Learning
I had reliable inputs for a perceptron and a clear output goal: difficulty scores per obstacle. But I lacked explicit target values for supervised learning or a reward signal for reinforcement learning. What I did have was an existing ordering of levels by difficulty.

If obstacle difficulty scores summed to level difficulty scores, then a genetic algorithm could optimize weights by scoring how closely the predicted level order matched the true order. A reinforcement learning setup, scored by ordering accuracy.

Before building the training environment, I prepared the feature and weight vectors, plugged in random values, and observed: 15% accuracy. Adding a random seed and watching accuracy vary confirmed the setup worked. Hand tuning weights achieved around 35%.

When the training loop ran ie splicing arrays, keeping top performers, I watched accuracy climb. It plateaued around 70%.

### Difficulty Curve
To generate levels, I combined similarity scoring with difficulty scoring. After training, I observed minimum and maximum level difficulty scores per game mode (Color Surge has several). Target obstacle counts were also known.

For determining a level's target difficulty, I considered training another perceptron mapping level number to difficulty score. But I noticed that easing functions, given min/max difficulty scores and level percentage, closely approximated real values. An easing function sufficed.

The generation pipeline: encode obstacles into character representations, compute and normalize feature vectors, encode levels, run inference to assign difficulty scores, order levels by score, extract min/max scores, compute target level's percentage, pass to easing function for target difficulty, use another easing function to select candidate obstacles (low difficulty obstacles for early levels, high difficulty for late levels), shuffle candidates, place obstacles until target difficulty is reached, compute the new level's representation, compare against existing levels. If similarity falls below threshold, keep it; otherwise, regenerate. Loop to generate entire modes.

### Accidental Hypernetwork
Meanwhile, I kept exploring neural networks. I wanted a working multilayer perceptron. The XOR function (the classic example of what single layer perceptrons cannot learn due to nonlinearity) became my test case. The perceptron failed; the MLP succeeded. I was fascinated. I must have fit that network to XOR twenty times.

Wanting a more interesting application, I returned to the PCG project. Rather than replacing the perceptron, I realized the MLP could learn values for the perceptron. I did exactly that.

Years later, I realized this fit the definition of a hypernetwork ie a network that generates weights for another network.

### K-Means Clustering
Before finalizing the PCG solution, I considered K-means clustering for difficulty scoring. I'd read about its ability to cluster min-max scaled feature vectors. Part of why I'd created them. One day, without reference code, I implemented K-means clustering, including K-means++ initialization, on my first attempt.

I expected clustering to split obstacles into three groups: easy, medium, hard. Instead, the network's difficulty outputs seemed scattered. When I increased K, something remarkable happened: visually similar obstacles clustered together with striking precision. I stared at the clusters, amazed.

Then I set it aside. It solved nothing for difficulty scoring.

### Loss Function
The neural network improved accuracy from 70% to about 85%, sometimes 87% with patience. Other modes maxed out entirely. I suspected the original Classic mode had an inconsistent difficulty curve, with hard levels occasionally being surprisingly easy.

I hypothesized that labeling obvious obstacles as easy, medium, or hard, and adding a classification term to the loss function, might help.

### Semi-Supervised Learning
The outcome was fascinating: the reinforcement learning problem partially became supervised learning through additional loss terms. Accuracy rose to 92%. More importantly, levels consistently matched their target difficulty. That extra signal went a long way. I considered the PCG project complete.

### Machine Learning Bots
I decided to test my skills further. A Color Surge executive was meeting a potential investor abroad. Many companies perceived increased value where machine learning existed. We already had ML for level generation.

I wanted to go further. Rovio Studios was combining procedural generation with ML bots to test levels. A natural next step for Color Surge.

I dove deeper into reinforcement learning, though I started simpler. If genetic algorithms could train networks to play Flappy Bird, they could work for Color Surge. I assumed this would be quick.

I was wrong.

I tried reinforcement learning with neural networks, then behavioral cloning via imitation learning.

Watching RL bots evolve was satisfying. When a bot first passed a level, I was startled. But performance plateaued. Bots couldn't surpass a certain score.

### Imitation Learning
I tried behavioral cloning: collecting observations (similar to Unity ML-Agents raycasting, but detecting color instead of object tags) paired with my actions while playing.

Getting this right required experimentation. At any moment, the optimal action depended on player velocity. Jumping might be correct at low velocity, wrong at high velocity. I didn't initially include velocity data. It took a year before I noticed Unity's ML-Agents documentation highlighted the importance of spatio temporal features; player velocity being one.

With velocity included, the imitation learning bot outperformed my RL bot. A pure feedforward network scored around 40 before losing. Remarkable for an early attempt.

### ML-Agents
I watched a video where someone discussing boids and spatial hashing trained LSTM networks using the NEAT algorithm. LSTMs seemed better suited for this temporal problem.

I switched to Unity ML-Agents. Color Surge runs on Unity, making it an obvious choice. LSTMs were available, and training used Proximal Policy Optimization (PPO).

The next five months were intense.

I used ~128 observations with 32 timesteps for the LSTM. I trained multiple bots. Frustratingly, they merely matched my imitation learning bot. Each LSTM learned to pass certain obstacles but failed on others. Subsequent training runs learned different subsets; never achieving general competence.

### Ensembles
I'd read that ensembling (averaging predictions from multiple models) consistently improves accuracy when individual models exceed 50% accuracy.

I trained multiple LSTMs independently, planning to average their outputs with learned weights.

### Accidental Sparse Mixture of Experts
But that's not quite what happened. Remember how K-means beautifully clustered obstacles by feature similarity? I decided to train one ensemble per cluster. An "ensemble of ensembles."

Each ensemble was referenced by cluster ID. At inference, observations were routed to the corresponding cluster. I'd considered clustering raw observations but liked using precomputed obstacle feature vectors. (Years later, I did try clustering raw observations.)

I precomputed clusters, loaded them into a lookup table, and routed via 1 nearest neighbor between obstacles and clusters. Input went to the matching ensemble.

I didn't realize it then, but this wasn't a traditional ensemble. I had accidentally created a sparse mixture of experts. I still called it an "ensemble of ensembles."

I viewed this as spatial partitioning in feature space: obstacles with nearby feature vectors were processed by the same ensemble; distant obstacles went to different ensembles. Later, learning about the manifold hypothesis, I concluded this architecture disentangled the manifold at a high level before networks learned lower dimensional representations.

Even so, some obstacles remained difficult, particularly those with randomly changing speeds. Spatio-temporal features, spurious correlations, and long range dependencies created hard limits. The bots couldn't reach superhuman performance.

One practical advantage: if a cluster gained new obstacles, only one ensemble needed retraining. Underperforming networks within an ensemble could be retrained independently. Ensembles trained in parallel, one per server. First locally, eventually on GPU servers.

### Emergent Generalization
Something important stood out. Individual LSTMs learned to pass limited obstacle subsets, failing on the rest. The sparse mixture of experts didn't have this problem.

I once heard that a model's success depends not just on accuracy but on generalization. When I deployed the sparse MoE bot in a new mode similar to Classic, it performed as though it had been trained on Classic. No previous architecture had achieved this. It could play five different modes.

### The GPT-4 Leak
I didn't realize I'd built a sparse mixture of experts until the GPT-4 architecture leak. Reading it to understand how advanced systems worked, I encountered the sparse MoE description in the context of multi head attention.

It clicked. My architecture shared high level similarities, though mine used LSTMs, not self attention, making them quite different in implementation.

### Theoretical Connections
Something finally clicked later. The "ensemble of ensembles" used weighted averages on outputs, similar to how multi head attention combines head outputs. Transformers use a down projection matrix (W_o) to project concatenated attention heads back to the original dimension. Averaging supposedly loses expressiveness due to equal weighting, making the learned projection preferable.

### Weighted Average
Examining my sparse MoE's outputs before averaging, I noticed: each network produces an output, and a single weight per network can modify behavior significantly; from one network dominating to all networks contributing equally.

While W_o matrices likely matter, I suspect they primarily learn sophisticated weighted combinations of concatenated inputs. It seems plausible that the entire W_o matrix could be replaced with one weight per attention head. Enabling fine tuning of experts with a few thousand weights while freezing the rest of the LLM.

I reach this conclusion because my network used exactly this setup, similar to the PCG project's perceptron. A hypernetwork learned these weights; original observation values could later refine the weighted average. These weights implemented "non naive bagging", a straightforward solution for fine tuning.

### Routing Mechanism
The router was a 1 nearest neighbor classifier (later re-designed for fine tuning). It used cached computations as an optimization. The best results came from modifying obstacle feature vectors to use only obstacle names before clustering. Names consisted of adjectives and nouns, which determined binding affinity between feature vectors and clusters.

Similar binding affinity appears in transformer style MoEs. The DeepSeek MoE paper discussed routers explicitly, mentioning "centroids" as expert identifiers. My approach loosely echoing theirs. OpenAI described their router only as "simple." DeepSeek's centroids were learned embeddings with binding affinity computed via cosine similarity (different from my Euclidean distance and precomputed centroids, but conceptually related)

### Fine Tuning
If binding affinity routes inputs to experts via cluster ID, then fine tuning a transformer MoE should resemble fine tuning an "ensemble of ensembles."

I learned weighted averages with a hypernetwork. Training the router was a linear projection enabling the 1NN classifier to route to any expert. In an LLM, the input to experts would still be original embeddings rather than modified ones.

Fine tuning became: learn a new router, plus new weights for the weighted average. This enables fundamentally different outputs per input. An expert trained primarily on one data distribution might vote on something superficially unrelated but subtly similar, improving next token prediction.

I like the idea of fine tuning with a small weight set for averages plus a somewhat small weight set for the router. In an RL setting, I'd train the router and the weighted averages.

### Conclusion
I never set out to build a Mixture of Experts system. I was just trying to make a little ball jump through spinning circles without the phone exploding.

But by repeatedly applying the same principle (decompose the problem space, then conquer each piece separately) whether through spatial hashes, clustering, or ensembling, I accidentally walked the exact same path that the most powerful language models in 2026 are now using at scale.

Sometimes the best discoveries aren’t planned. They’re just the natural consequence of refusing to accept “impossible” as an answer.
