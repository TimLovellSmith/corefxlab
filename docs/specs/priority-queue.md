# Priority queue proposal

## User data

Before anything is proposed, let's start with what can be prioritized...

### Input types

We want to be able to support all of the following configurations:

1. User data *is separate* from the priority (two physical instances). 
2. User data *contains* the priority (as one or more properties). 
3. User data *is* the priority (implements `IComparable<T>`).
4. Rare case: priority is obtainable via some other logic (resides in an object
different from the user data).

In this entire document, I will refer to the cases above with **(1)**, **(2)**,
**(3)**, and **(4)**. We will also refer to case **(1)** as 'extrinsic' priority, 
and all the other cases as 'intrinsic' priority.

### Ideal solution

Obviously, our solution should be flexible enough to (respectively):

1. Simply accept two separate instances. The user should not be forced to create
a wrapper class for the two types only because of our API limitations.
2. Accept an element that already has the priority in it, without duplication
(no copying).
3. Be able to use `IComparable<T>` and don't expect an additional priority.
4. Be able to execute some additional logic that retrieves the priority for a
given element.

### Our approach

In order to be able to consume all of that, we need two types of priority
queues:

* `PriorityQueue<TElement, TPriority>` (this handles the extrinsic priority scenario, **(1)**)
* `PriorityQueue<T>`, (this handles all the intrinsic priority scenarios)

## `PriorityQueue<TElement, TPriority>`

```csharp
public class PriorityQueue<TElement, TPriority> :
    IEnumerable,
    IEnumerable<(TElement element, TPriority priority)>,
    IReadOnlyCollection<(TElement element, TPriority priority)>
{
    public PriorityQueue();
    public PriorityQueue(IComparer<TPriority> comparer);
    public PriorityQueue(IEnumerable<(TElement, TPriority)> collection);
    public PriorityQueue(IEnumerable<(TElement, TPriority)> collection, IComparer<TPriority> comparer);

    public IComparer<TPriority> Comparer { get; }
    public int Count { get; }

    public bool IsEmpty { get; }
    public void Clear();
    public bool Contains(TElement element);
    public bool Contains(TElement element, out TPriority priority);

    public void Enqueue(TElement element, TPriority priority);

    public (TElement element, TPriority priority) Peek();
    public bool TryPeek(out TElement element, out TPriority priority);
    public bool TryPeek(out TElement element);

    public (TElement element, TPriority priority) Dequeue();
    public bool TryDequeue(out TElement element, out TPriority priority);
    public bool TryDequeue(out TElement element);

    public bool Remove(TElement element);
    public bool Remove(TElement element, out TPriority priority);

    public IEnumerator<(TElement element, TPriority priority)> GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator();

    public IEnumerable<TElement> Elements { get; }
    public IEnumerable<TPriority> Priorities { get; }

    public struct Enumerator : IEnumerator<(TElement element, TPriority priority)>
    {
        public (TElement element, TPriority priority) Current { get; }
        object IEnumerator.Current { get; }
        public bool MoveNext() => throw new NotImplementedException();
        public void Reset() => throw new NotImplementedException();
        public void Dispose() => throw new NotImplementedException();
    }
}
```

### Scenarios Covered

The user has some data and priority separated — **(1)**:

```csharp
var userData = "this is a string";
var priority = 5;
```

So simply:

```csharp
var queue = new PriorityQueue<string, int>();
queue.Enqueue(userData, priority);
...
// something changed, so we need to update priority
queue.UpdatePriority(userData, newPriority);
```

## `PriorityQueue<T>`

```csharp
public class PriorityQueue<T> : IQueue<T>
{
    public PriorityQueue();
    public PriorityQueue(IComparer<T> comparer);
    public PriorityQueue(IEnumerable<T> collection);
    public PriorityQueue(IEnumerable<T> collection, IComparer<T> comparer);

    public IComparer<T> Comparer { get; }
    public int Count { get; }

    public bool IsEmpty { get; }
    public void Clear();
    public bool Contains(T element);

    public void Enqueue(T element);
    public void NotifyPriorityChanged(T element);
    public void NotifyPriorityChanged(IEnumerable<T> elements);

    public T Peek();
    public T Dequeue();
    public bool Remove(T element);

    public bool TryPeek(out T element);
    public bool TryDequeue(out T element);

    public IEnumerator<T> GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator();

    public struct Enumerator : IEnumerator<T>
    {
        public T Current { get; }
        object IEnumerator.Current { get; }
        public bool MoveNext() => throw new NotImplementedException();
        public void Reset() => throw new NotImplementedException();
        public void Dispose() => throw new NotImplementedException();
    }
}
```

### Scenarios Covered

#### (2)

Custom class with a priority inside:

```csharp
public class MyClass
{
	public double Priority { get; }
}
```

The user defines their own comparer, for example:

```csharp
var comparer = Comparer<MyClass>.Create((a, b) =>
{
    return a.Priority.CompareTo(b.Priority);
});
```

And simply uses our priority queue:

```csharp
var queue = new PriorityQueue<MyClass>(comparer);

var x = new MyClass();
queue.Enqueue(x);
...
// something changed, so we need to both update the priority, and then tell the queue about the priority change, otherwise priority queue order invariants will be broken
x.Priority = newPriority;
queue.NotifyPriorityChanged(x);

```

#### (3)

Already comparable type:

```csharp
public class MyClass : IComparable<MyClass>
{
    public int CompareTo(MyClass other) => /* some logic */
}
```

Then simply call the default constructor (`Comparer<T>.Default` is assumed):

```csharp
var queue = new PriorityQueue<MyClass>();
```

_If_ the object is mutable, and any property of the object changes which would cause the Comparer to start returning different results, then it is necessary to call `NotifyPriorityChanged()` so that the priority queue can be kept consistent;

```csharp
mutatedElement.SomeProperty = newValueWhichAffectsComparison;
queue.NotifyPriorityChanged(mutatedElement);
```


#### (4)

Priority for `MyClass` is obtainable from some other objects, for example a
dictionary. It is done analogically to **(2)**, simply by some custom logic in
the comparer. If the state of other objects (e.g. the dictionary) changes in a way which affects priority, then again it is necessary to notify the priority queue of the change.

## Notes

To both priority queues:

* If the `IComparer<T>` is not delivered, `Comparer<T>.Default` is summoned.
* `Peek` and `Dequeue` throw an exception if the collection is empty.
* `TryPeek` and `TryDequeue` only return false.
* `Remove` returns false if the element to remove is not found.
* `Remove` removes only the first occurrence of the specified element.

_If_ the object's priority is mutable, and any property of the object changes which would cause the Comparer to start returning different results for any objecst, then you must call `NotifyPriorityChanged()` passing all affected elements, so that the priority queue can be kept consistent;

```csharp
mutatedElement.SomeProperty = newValueWhichAffectsComparison;
queue.NotifyPriorityChanged(mutatedElement);
```

## `IQueue<T>`

With the design above, we can address some voices regarding the introduction of
`IQueue<T>`:

```csharp
public interface IQueue<T> :
    IEnumerable,
    IEnumerable<T>,
    IReadOnlyCollection<T>
{
    int Count { get; }

    void Clear();

    void Enqueue(T element);

    T Peek();
    T Dequeue();

    bool TryPeek(out T element);
    bool TryDequeue(out T element);
}
```

### Notes

* Only `PriorityQueue<T>` would implement this interface. (Why?)
* `IsEmpty` does not need to be a member of `IQueue<T>`, since you can just use Enumerable.Empty().

2. In priority queues, there is `Peek`, and `TryPeek`. There is also `Dequeue` and
`TryDequeue`. There does not need to be `Remove` and `TryRemove` following the same
pattern - consistent with e.g. ISet<T>, Remove can always return a bool with no other returned value. Whereas the Peek/TryPeek pattern and Remove/TryRemove are a way of working around the problem of having a conditionally returned value.

## Open questions

1. Do we really need `IQueue<T>`?
2. What are  the performance guarantee that UpdatePriority and NotifyPriorityChanged should give?
3. Should Remove, and UpdatePriority be targetable at unique nodes
instead of *whichever is found first* using some kind of handle? This would require us to use some sort
of a handle:

```csharp
void Enqueue(TElement element, TPriority priority, out object handle);

void Update(object handle, TPriority priority);

void Remove(object handle);
```

It might be unnecessary because if the elements are 'equal' according to the equality comparer used for searching, that should imply it is fine to remove either one of them. So should there e.g. be constructor overloadsfor specifying a custom equality comparer for that purpose? This is confusing, because the priorityComparer and equalityComparer are comparing totally different things (elements, that have priority on one hand, versus priorities on the other hand!!)

    public PriorityQueue(IEnumerable<T> collection, IComparer<T> comparer, IEqualityComparer<T> equalityComparer);


4. Do we want to provide an interface for priority queues in the future? If we
release two `PriorityQueue` classes, it may be hard to create an interface for
them.
5. If we don't want to add the concept of handles in our priority queues, we are
basically locking our solution on less efficient and less correct support for
updating / removing arbitrary elements from the collection (problem in Java).
It is additionally more problematic if we don't add a proper interface. A
solution could be to add a proper support for the heaps family (`IHeap` +
possibility of various implementations) in `System.Collections.Specialized`.
   * Developers could write their third-party solutions based on a single,
   standardized interface. Their code can depend on an interface rather than an
   implementation (`PriorityQueue<T>` or `PriorityQueue<T, U>` — which to choose
   as an argument?).
   * If such a functionality is added to `CoreFXExtensions`, there would be no
   common ground for third-party libraries.
   * Decision where this would eventually land (`CoreFX` or not) directly
   impacts whether missing features in our support for priority queues are an
   issue or not. If we add heaps to `System.Collections.Specialized`, priority
   queues can be lacking more power and enhanceability. If we don't, there is an
   issue.
