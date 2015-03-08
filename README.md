# Enhanced Vector and Deque Containers
####Proposal for Boost GSoC '15


##Introduction
Taken from the [Boost GSoC '15 wiki:](https://svn.boost.org/trac/boost/wiki/SoC2015#a5.Enhancedvectoranddequecontainers)
> For certain applications the standard containers std::vector and std::deque are not too easy to work or may induce unnecessary overhead. For example, std::vector lacks an O(1) time push_front and std::deque allows you no way of controlling the size of the underlying segments. This limits their usability in certain contexts, especially if you want to have optimal performance.

Certain applications (for example, intrusive containers, as noted [here](https://svn.boost.org/trac/boost/wiki/soc2009#Boost.Devector)) also require guarantees such as iterator stability, which is not provided by **std::vector**. **std::deque** is basically implemented as an array of arrays (segments), but does not allow any control over the underlying segments. The deque also does not allow reserving of capacity beforehand a la ``` std::vector::reserve()``` Furthermore, iteration over a deque could be faster using [segmented iterators][segmented-iterators]. These missing features somewhat limit the use of these standard containers in performance-intensive applications. Thus, there is a need for improved versions of these containers without these shortcomings. I would like to propose two alternate containers: **devector** and **controlled_deque** (name subject to change). These are explained in greater detail below.

## The devector container
Reference stability is not a new concept; a reference-stable of std::vector in fact already exists within Boost ([**stable_vector**][stable-vector]). However, **stable_vector** still lacks an _O(1)_ time ```push_front()```. Qt, however, offers the [**QList**](http://doc.qt.io/qt-5/qlist.html#details) class, which offers [amortized constant-time insertion](doc.qt.io/qt-5/containers.html#algorithmic-complexity) at both the beginning and the end of the vector along with reeference stability. It achieves this by preallocating space at both the front and the back of the underlying array. But the **QList** class lacks the ``` shrink_to_fit()``` function, which reduces the capacity of the vector to match its size. Thus the **QList**'s internal array never shrinks, and this may lead to wasted memory.

My proposed **devector** container is mostly the same as Boost's **stable_vector**, with the preallocated memory features borrowed from Qt's **QList**. Thus, **devector** is reference-stable and also offers _O(1)_ time insertion at the front.A simplified class definition of **devector** is shown below:

```
template <class T, class Allocator = std::allocator<T>>
class devector
{
private:
    struct node
    {
        node* up;
        T data;
    };

public:
    typedef T value_type;


    // Member functions of std::vector
    // ...

    // Proposed new member functions
    void push_front(const value_type& val);
    void push_front(value_type&& val);

    void pop_front();

    template <class... Args> void emplace_front(Args&& args);

    // The reserve function will be slightly modified.
    // See explanation below.
    void reserve_front(size_type n);
    void reserve_back(size_type n);
    void reserve(size_type n);

private:
    node* _array;
    node* _start;
    size_type _size;
    size_type capacity;
    // ...
};
```

###Member function summary
1. ``` void push_front(const value_type& val)```  
 This inserts the given element at the front of the vector. It triggers a reallocation if there is no more space at the front.

2. ``` void pop_front()```  
 Destroys the element at the front of the vector.

3. ``` template <class... Args> void emplace_front(Args&& args)```  
 Constructs an element at the front of the vector with ``` args``` forwarded. it triggers a reallocation if there is no more space at      the front.

4. ``` void reserve_front(size_type n)```  
 Reserves enough memory for ``` n``` elements at the front of the vector by reallocating the underlying array.

5. ``` void reserve_back(size_type n)```  
 Reserves enough memory for ``` n``` elements at the back of the vector by reallocating the underlying array. This function is similar to ``` std::vector::reserve()```.

6. ``` void reserve(size_type n)```  
 Reserves enough memory for ``` (n / 2)``` elements at the front of the vector, and ``` ceil(n / 2.0f)``` at the back of the vector. This function reallocates the underlying array only once and is more efficient than calling ``` reserve_front()``` followed by ``` reserve_back()```.

###Implementation details
Since **devector** is an array of pointers, it will need a node type **devector::node** to store elements. The nodes will have element data and an 'up' pointer, as in [**stable_vector**][stable-vector]. ``` _start``` will point to the beginning of the array of pointers to nodes. This array will begin somewhere in the middle of the underlying array, hence allowing for elements to be inserted at the front without relocation. A past-the-end pointer can be obtained by ``` _start + _size```. Iterators will behave similarly to those of [**stable_vector**][stable-vector].

###Issues/shortcomings:
* The elements are not contiguous.
* Storing of nodes rather than raw data items incurs some overhead.

##The controlled_deque container
The main new feature of the **controlled_deque** container is the segmented iterator. Segmented iterators (as described [here][segmented-iterators]) are iterators which expose the underlying segments of the deque. **controlled_deque** will also allow the user to specify the size of the segments in advance. It will also have ``` reserve()``` functions similar to those in **devector**. Thus, **controlled_deque** puts more control in the hands of the user. A simplified class definition is given below:

```
template <class T, class Allocator = std::allocator<T>>
class controlled_deque
{
public:
    struct local_iterator
    {
        T* _pointer;

        // This iterator traverses a single segment

        // Operator overloads as in conventional iterators
        // ...
    };

    struct segment_iterator
    {
        T** _pointer;

        // This iterator traverses the array of segments

        // Return local_iterators to the beginning and end of the current segment
        local_iterator begin();
        local_iterator end();

        // Operator overloads as in conventional iterators,
        // except operator*(), which is equivalent to calling
        // begin().
        // ...
    };

    struct segmented_iterator
    {
        // This iterator composes both segment and local iterators

        segment_iterator segment;
        local_iterator local;

        // Operator overloads as in conventional iterators
    };

    // Member functions as in std::deque
    // ...

    // Proposed new member functions

    controlled_deque(size_type segment_size, size_type number_of_segments);

    segmented_iterator begin();
    segment_iterator end();

    void reserve_front(size_type n);
    void reserve_back(size_type n);
    void reserve(size_type n);

private:
    const size_type SEGMENT_SIZE;
    // ...
};
```

###Member function summary
1. ``` controlled_deque(size_type segment_size, size_type number_of_segments)```  
 This new constructor allows the user to pass the segment size. A number of segments to be preallocated can also be passed.

2. ``` segmented_iterator begin()``` and ``` segmented_iterator end()```  
 These return segmented iterators to the beginning and past the end of the deque respectively. They are intended to serve as replacements for **std::deque**'s existing ``` begin() and ``` end()``` functions.

3. ``` reserve(size_type n)```, ``` reserve_front(size_type n)``` and ```reserve_back(size_type n)```  
 These reserve functions behave similarly to those in **devector**, except that they reserve entire segments at a time. The number of segments to be reserved is calculated as ``` ceil(static_cast<float>(n) / SEGMENT_SIZE)```, where ``` n``` is the number of elements to be reserved.

###Implementation details
Both ``` local_iterator``` and ``` segment_iterator``` behave like traditional random-access iterators. The only difference is in their value types. They both have operator overloads as in normal iterators and hence can be used similarly. ``` segmented_iterator```, in addition to exposing the corresponding ``` segment_iterator``` and ``` local_iterator```, also supports the random-access iterator concept and has a value type of ``` controlled_deque<T>::value_type``` and hence can be used just like ``` std::deque<T>::iterator```. Thus **controlled_deque** is fully STL-compliant. Smarter algorithms, however, will leverage the provided ``` segment_iterator``` and ``` local_iterator``` members for faster and more controlled iteration.

###Issues/shortcomings
There are no trade-offs or shortcomings as such. **controlled_deque** can be used as a drop-in replacement for **std::deque**.

##Deliverables
* **devector** along with its iterator and the described member functions
* **controlled_deque** along with its segmented iterator and the described member functions
* Tests and documentation for **devector** and **controlled_deque**



[stable-vector]: http://www.boost.org/doc/libs/1_57_0/doc/html/container/non_standard_containers.html#container.non_standard_containers.stable_vector

[segmented-iterators]: http://lafstern.org/matt/segmented.pdf
