Title: Particle Counting
Date: 2022-09-04 10:20
Category: Physics, Algorithms

<script type="text/javascript"
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.0/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
<script src='./scripts/public/jquery-1.11.2.min.js'></script>
<script src='./scripts/main.js'></script>


I've lately been collaborating loosely with a physics group at the University of Iowa because they have an interesting problem that struck kind of close to something I've been working with on and off again for awhile. This group is doing some neat things with filming lab scale dusty plasma, which, glibly, are plasmas that have, in addtion to electrons and nuclei (like, argon in a lab, or hydrogen in space) also contain larger particles (like polystyrene in the lab, or grains of sand in the asteroid belt). These dust particles effectively become very massive extra species added to the complex dance that is plasma dynamics. Of chief interest to the experimentalist is that they have the added benefit of being big enough and slow enough to capture on off-the-shelf video equipment.

How is that a big deal? Well, besides new opportunities for the likes of David Attenburrough, filming the plasma in situ permits a variety of measurements that would make <a href="https://en.wikipedia.org/wiki/William_Thomson,_1st_Baron_Kelvin">Lord Kelvin</a> break down in tears. The pipe line works as follows, take a video of the plasma, break the video down into individual frames.

<canvas id='small-sample-blue' width=200 height=200></canvas> +
<canvas id='small-sample-red' width=200 height=200></canvas> =
<canvas id='small-sample-both' width=200 height=200></canvas>

Next, make use of well researched computer vision algorithms to do centroid detection of individual particles, turning a still image into a list of x-y pairs telling you where your particles are.

$&blue=[(x_0,y_0),(x_1,y_1),(x_2,y_2),...]&$ and $&red=[(x_0,y_0),(x_1,y_1),(x_2,y_2),...]&$

We'll refer to those as text-frames as they are a text file that contains the most important parts of these image frames. Then, take two adjacent text-frames, and stitch them together, finding where particle A is in both the first frame and the second, producing a path. Now, ideally each x-y pair in the first text-frame will have a corresponding pair in the second text-frame, because conservation of mass is generally a thing.

$&paths=[
      ((x_{b0},y_{b0}),(x_{r1},y_{r1}),...),
      ((x_{b1},y_{b1}),(x_{r7},y_{r7}),...),
      ((x_{b2},y_{b2}),(x_{r84},y_{r84}),...),
      ...
      ]&$

However you have to have a little give, because the field of view of the camera does not encapsulate the entire experiment. This will impact some of the particulars of an implementation, but not relevant to the nature of this specific discussion which is about the more high level approach on the matter.

So, the obvious thing to do is, if we have one list, which has in it a pair for a member of another list as decided by some metric (say, a euclidean distance less than 5 pixels), all we have to do is just go through each member of the candidates and pair them up one at a time. This is very straightforward and understandable. But it is also somewhat labor intensive, say we start off with subsets with only about 100 particles. We take each of those blue particles, check each of the red particles for which one is closest, and say they make a path. So for each of the hundred blues, we'll need to look at all hundred of the reds, because as a simple list, we can't really tell if one is closer than any of the others. Or about 10000 comparisons over all, 100 X 100.

Ok, you might say. That's not so bad, the smaller sets with about a hundred particles or so ran in about a hundredth of a second, choosing a tiny number at not quite random. How bad could it be?

<img src='./images/cW__0001.png'/>

Merciful Turing, these slides have like, thousands of particles in them! Ok, ok, calm down, that just means that if we have 30 times as many particles, we have 900 times as much work, that still only means it takes about 9 (.01 X 900= 9) seconds to stitch together 2 text-frames. Oh, wait, some of these films have thousands of frames? So .01 X 900 X 5000= 45000 seconds = 750 minutes = 12.5 hours! It takes a few seconds to film it, and then the better part of a day to start interpreting the data. Is there a better way to go about doing this?

There is! This problem is effectively an iterated nearest neighbor search, which has a variety of clever solutions for arbitrary situations where you have an unknown number of dimensions and no constraints on distribution. However, we're not in arbitrary space with the most general case possible. We're in a two dimensional space, so implementing a quad tree would provide very good improvements. But we don't need to figure out how to implement a quad tree to get most of the benefits it would offer. Look back at that picture,

<img src='./images/cW__0001_mini.png'/>

Everyone of those dots has two features that will simplify the problem of stitching them together. First, they have mass, so if we film at a high enough frame rate, we can be confident they won't move more than a certain distance. And secondly and more importantly, they have charge. This means they naturally want to maintain a relatively even spacing between each other, like atoms in a salt crystal. Quad trees are great for storing proximity data in systems that are barely structured, like when you're doing an n-body simulation and you want to use the <a href="https://en.wikipedia.org/wiki/Barnes%E2%80%93Hut_simulation">Barnes Hut Algorithm.</a> But overkill with such smoothly distributed bodies.

These features allow us to have a much simpler data structure to store candidates in. Instead of a fully dynamic tree, we just have a nested collection of arrays, you can think of them as chessboard-style collection of buckets. So now we have two lists (first and second), and a big grid of empty buckets.

<canvas id='empty-bins' width=200 height=200></canvas>
    

We now take the second list, figure out what bucket each particle belongs to and put it there.


<canvas id='filled-red-bins' width=200 height=200></canvas>


After this, we can take each particle from the first list, figure out what bucket <em>it</em> belongs to, but instead of putting it there, we just look at what's in it, and the buckets adjacent to it. In the following example we would only have to compare the blue particle to two red ones, instead of the entire collection of red ones.


<canvas id='blue-check-bins' width=200 height=200></canvas>


Ideally we would only find one particle, but in practice it might be 3 or 4 or if we made our buckets grossly too large, hundreds. But given the physical constraints of the data we're looking at, we are much more likely to have between 1 and 10 particles.




This is just one example where it's possible to take advantage of structure in data to make efficiency gains, much like how a sorted array can have a binary search executed on it while a randomized array can't. When looking for speed ups, it behooves us to think about what we're trying to accomplish and what we have to work with in the beginning. Often times there's structures in what we're treating as random data, and sometimes we'll over complicate. When I first thought of using a data structure to speed things up, I knew a quad tree would make the looks up faster, but I didn't know how to implement it. Then I realized that they're basically in a hexagonal lattice, and I <em>did</em> know how to map a hexagonal lattice into a rectangular one, but waiting on it I realized the difference between the two would have been slight compared to just using the buckets as is.

<h3>Exploration</h3>
So it's possible to get considerable speed ups with an appropriate data structure which filters out how many other particles we want to check. But how do we decide how much to filter out? Lets say for the sake of argument that setting aside the memory to make a bucket takes about as much time and/or computational effort as checking the distance between two particles, this is an oversimplification, but increased accuracy wouldn't buy as that much more understanding in this case. Below is a little exhibit of how too large of buckets means we don't get much time savings on checking particles, but too tiny of buckets means we spend too much time making the buckets. The lime green bar is how much work is being done checking particles, and the light blue bar is how much work is being done making buckets. Fiddle around with the slider to see how the trade off works. In general less work is better.
<div id='demo-area'>
  <div id='visual'>
    <canvas id='demo-zone' width=400 height=300></canvas>
    <form id="controls">
      <input type='range'
      id='bin-spacing-controls'
      min='3'
      max='100'>
      </input>
    </form>
  </div>
  <div id='graphs'>
    <canvas id='charts-zone' width=200 height=300></canvas>
    <br>
    <span class='left'>Bucket work</span>
    <span class='right'>Distance work</span>
  </div>
</div>
  </div>
  <div>
    <h3>Further Reading</h3>
    <a href="https://en.wikipedia.org/wiki/Nearest_neighbor_search">The wikipedia article on Nearest Neighbor Searching</a>
    <br>
    <a href="https://en.wikipedia.org/wiki/Fixed-radius_near_neighbors">The wiki article fixed-radius nearest neighbors</a> which lists on it's 3rd citation an implementation on the GPU for maximum through-put.
  </div>
</body>
