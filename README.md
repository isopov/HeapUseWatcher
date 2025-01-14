HeapUseWatcher
===========

A simple tracker of non-ephemeral heap use and live set levels, which may be useful
for making application logic decisions that react to heap occupancy (in e.g. choosing
when and how much application-managed in-memory cached contents to keep or evict).

This tracker leverages a trivially simple observation: In any collector, generational
or not, the stable set of actually live objects will eventually make it into the oldest
generation. We focus our reporting on the oldest generation, and completely ignore
all other use levels and activity.

Reasoning: Any memory use in other, younger generations, regardless of their shapes and
parts, is temporary, and any non-ephemeral objects that reside there will eventually get
promoted to the oldest generation and be counted. While it is possible to temporarily
miss such objects temporarily by ignoring the younger generations, their impact on the
oldest generation will eventually show up.

Ignoring the promotion of yet-to-be-promoted objects (when considering the oldest
generation use against the maximum capacity of that same oldest generation) is logically
equivalent to ignoring the allocations of yet-to-be-instantiated objects in the heap
as a whole. It is simply an admission that we cannot tell what the future might hold,
so we don't count it until it happens.

Establishing the current use level (as opposed to the live set) in the oldest
generation is a relatively simple thing. But for many purposes, estimation of the
live set would be more useful generally. Making logic choices purely on the
currently observed level (which includes any promoted, now-dead objects that have
not yet been identified or collected) can often lead to unwanted behavior, as
logic may conservatively react to perceived use levels that are not real, and
indicate a "full or nearly full" heap when plenty of (not yet reclaimed) room
remains available.

Live set estimation:
Due to the current limitations of the spec'ed memory management beans available in Java
SE (up to and including Java SE 13), there is no current way to establish an estimate
of the "live set" in the oldest generation without use of platform-specific (non-spec'ed)
MXBeans, which provide access to information about use levels before and after collection
events. While using such MXBeans does provide additional and better insight into live-set
levels, a portable approximation of the same can be achieved by watching the current use
levels in the oldest generation, and reporting the most recent local minima observed as
live set estimations [A local minimum in this context is the lowest level observed between
two other, higher levels, with some simple filtering applied to avoid noise].

This class provides multiple forms of use:

A. The runnable HeapUseWatcher class can be
launched and started as a thread, which will independently
maintain an updated model of the non-ephemeral heap use.

B. One can directly use HeapUseWatcher.NonEphemeralHeapUseModel
and call its updateModel() method periodically to keep the model
up to date.

C. The class can be used as java agent, to add optional
reporting output to existing, unmodified applications.

An exmple of using HeapUseWatcher as a java agent (it is used to
monitor heap use and live set in a [HeapFragger](https://github.com/giltene/HeapFragger)
run. HeapFragger which is a simple active excercizer of the heap and
presents a nice moving target for HeapUseWatcher to track):

% java -Xmx1500m -XX:+UseG1GC -javaagent:HeapUseWatcher.jar="-r 1000" -jar HeapFragger.jar -a 3000 -s 500

See the class JavaDoc for org.heaputils.HiccupUseWatcher for use cases A and B above.