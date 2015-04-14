---
layout: post
title: "Building skyscreen"
date:   2015-04-14 22:03:00
---

Some friends and I have decided to build a skyscreen. Imagine a four meter diameter circular screen of dense LEDs. That's skyscreen.

And I'm working on the software.

The problems
------------

There are a couple of issues with this, software wise, but mostly the revolve around performance and ease-of-programming. On the performance side, we'd love to run this on a raspberry pi, and they aren't fast. On the ease-of-programming side, I don't think it's a good idea to write these sorts of complex image manipulations in C, or C++, or in any low level language. I doubly-don't think we should allow a crash in the patterning code to cause a crash in the display code.

Here's what I ended up with:

<iframe width="740" height="555" src="https://www.youtube.com/embed/dPbLuqdxgGE" frameborder="0" allowfullscreen></iframe>

The solution
------------

The first thing that sprang to mind was to use a shared area of memory for communication. This would let us be language agnostic and it would (somewhat) isolate faults. It also worked out very neat, so I think it's the right way to go.

So, in the end, I used a memory mapped file for both communication and locking. The operating system should cache the changes, rather than flushing to disk, so we aren't constrained by the HDD speed (or the SD card speed, which is even slower).

Step one was to just write a very simple python class that would return the mapped array. This would turn out to be far far too slow to be useful, but it was a great reference implementation, which I could use to find issues with the other implementations.

Here's a commented version of the ```writer``` implementation, which you can find [here](https://github.com/ririw/skyscreen/blob/master/skyscreen_core/mmap_interface.py)
{% highlight python %}
	class MMAPScreenWriter(BaseMMapInterface, skyscreen_core.interface.ScreenWriter):
		file_mode = os.O_CREAT | os.O_RDWR

		def __init__(self, shared_file):
			super(MMAPScreenWriter, self).__init__(shared_file)

		def initialize_file(self):
			self.shared_handle = os.open(self.shared_file, self.file_mode)
			assert self.shared_handle, 'could not open: %s' % self.shared_file
			assert os.write(self.shared_handle, '\x00' * self.array_size) == self.array_size
	
		def __enter__(self):
			assert self.shared_memory is None, 'cannot open shared mem twice'
			self.initialize_file()
			self.shared_memory = mmap.mmap(self.shared_handle, self.array_size, mmap.MAP_SHARED, mmap.PROT_WRITE)
			return self.shared_memory
{% endhighlight %}

As you can see, it just returns the mmapped array. It's too simple to get wrong.

> This would turn out to be far to slow to be useful, but it was a great reference implementation.

Performance problems
--------------------
The above approach is much too slow. Not even close to the 30FPS target, even on my laptop, let along a pi. The next step was to investigate numpy and cython. I never worked out how to avoid hitting the array bounds checks in cython, so it wasn't super helpful, but I did work out how to get good performance with numpy.

Numpy has a module specifically for memory mapping in files and treating them as arrays. This is _almost_ perfect. It lets me use all the cool numpy tools to write my code, and its simple to use. I was really worried I'd have to go into cython for everything, or re-write it all in C++.

You can find this code [here](https://github.com/ririw/skyscreen/blob/master/skyscreen_core/memmap_interface.py)

{% highlight python %}
class NPMMAPScreenWriter(BaseMMapInterface, skyscreen_core.interface.ScreenWriter):
	file_mode = os.O_RDWR

	def __init__(self, shared_file):
		super(NPMMAPScreenWriter, self).__init__(shared_file)

	def __enter__(self):
		assert self.shared_memory is None, 'cannot open shared mem twice'
		self.shared_memory = np.memmap(self.shared_file,
						  dtype=np.byte,
						  mode='w+',
						  shape=(self.screen_vane_count*self.screen_vane_length*3))
		return self.shared_memory
{% endhighlight %}

In some ways, this is simpler than the mmap approach. It was also really easy to test - just set up a mmap reader with a numpy writer and see if the input matches the output. Similarly, you can switch the two and have a numpy reader and mmap writer. Check out [the tests](https://github.com/ririw/skyscreen/blob/master/tests/test_memmap.py)

This easily hit the FPS goal on my laptop, but there were still problems. In particular, it was very difficult to tweak small areas of the screen and still be performant, because of the python array overhead. 

Theano
------
I've been using theano both at home and at work for machine learning. It's a great approach to numerical code. You develop your numerical program in almost symbolic form, and then when it's executed it's optimized and "compiled" to a higher perforamnce version. It even supports GPU rendering, and if I get my hands on a Jetson TK1 (which has an NVIDIA GPU on it), I'll see if the system runs better than on the PI by using the GPU.

Anyway, theano made this all very easy. Even for complex expressions, or 'map' style expressions, where I wanted to evaluate over every pixel on the screen, these were easy to make perforant, and only slightly more complex than an equivalent numpy version. For example, here is the function that is mapped over every pixel to get an expanding droplet effect:

{% highlight python %}
# Step is already in scope. vane, px and color are the vane, 
# the pixel and the color for the array cell being operated on
def draw(vane, px, col): 
	ring = Screen.screen_vane_length-(step/3 % Screen.screen_vane_length)
	ring_dist = T.maximum(0, float(Screen.screen_vane_length)/(ring-px))
	circ_adjustment = vane
	return T.clip(ring_dist, 0, 255)
{% endhighlight %}

See the [theano examples](https://github.com/ririw/skyscreen/blob/master/screens/theano_examples.py) for some more of this sort of stuff

Concluding remarks
------------------
I'll keep this simple:

- Python is slow. There's so much indirection and overhead, it's just not good for performance code
- But it's libraries are fast! And you probably want them anyway (at least if you're doing numerical code). 
  This probably makes it the best bet for writing fast numerical code with good productivity, simply because
  there's so much out there already
- And sometimes you still need to fall back to C or C++. In the end, the rendering code is in C++, not python.
- Having a good, clean interface is a great way to simplify your program. Using a memory mapped file meant it 
  was easy to keep my programs separate and reduced complexity.
- And having a good clean interface means that if I need to move to some other language for a particular use
  case, that's easy.
- Skyscreen is awesome.
