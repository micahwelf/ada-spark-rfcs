- Feature Name: Variable/Constant Range Indexing
- Start Date: 2020-03-22
- RFC PR: 
- RFC Issue: 

Summary
=======

Currently Variable Indexing performs a vital function for readable and efficient
code.  In contexts where it makes a vector indistiquishable from an array,
there is still one major difference: index slicing by range.  I propose this
be provided in the same manner the aspects Variable_Indexing  and 
Constant_Indexing are provided, such as Variable_Range_Indexing and
Constant_Range_Indexing.


Motivation
==========

When accessing a slice of a private, tagged vector we can read a slice easy enough by copying
it's contents with a slice function, but if we want a more efficient way of
reading *and writing* a slice we would need direct access to the array component of the private
record.  The only way this could be done now is to set up a loop to access
each element of that component, in the range, individually. The overhead of
setting up the access to the elements could be consolidated and become more
efficient, as well as more readable, if it were done once, allowing direct
read *and write* to the elements.  It would also start the range checking of
the complete range given before any reading or writing takes place.
Just as Variable_Indexing saved us from a lot of unnecessary operations and
visual noise, Variable_Range_Indexing can save us from the same and provide
new ways to access internals of an object without removing its privacy.

Guide-level explanation
=======================

Setting up Variable_Indexing requires a lookup function and a returned 
Reference_Type that accesses a single element.  To setup a 
Variable_Range_Indexing aspect, you create a function that takes exactly
three arguments: first, of the associated tagged type, second the low
end of the range, and third, the high end of the range.  Next you need
a Reference_Range_Type that refers to an array object that will be, internally,
a renamed portion of the internal private component array.

    generic
    
      type Index_Type is (<>);
      
      type Element_Type is private;
      
      type Element_Array is array (Index_Type) of Element_Type;
      
    package Container_Vectors is
    
    type Reference_Type (Element : not null access Element_Type) is null record
     with Implicit_Dereference => Element;
    
    type Constant_Reference_Type (Element : not null access constant Element_Type) is null record
     with Implicit_Dereference => Element;
    
    type Reference_Range_Type (Slice : not null access Element_Array) is null record
     with Implicit_Dereference => Slice;
    
    type Constant_Reference_Range_Type (Slice : not null access constant Element_Array) is null record
     with Implicit_Dereference => Slice;
    
    type Vector is tagged private
    
     with 
     
      Variable_Indexing => Reference,
      
      Constant_Indexing => Constant_Reference,
      
      Variable_Range_Indexing => Reference_Range,
      
      Constant_Range_Indexing => Constant_Reference_Range;
    
    ...
    
    function Reference_Range (Subject : Vector;
    
                              First : Index_Type;
                              
                              Last  : Index_Type)
                              
                              return Reference_Range_Type;
    
    function Constant_Reference_Range (Subject : Vector;
    
                                       First : Index_Type;
                                       
                                       Last  : Index_Type)
                                       
                                       return Constant_Reference_Range_Type;
    
    private
    
    ...
    
    end Container_Vectors;

When you go to read from or write to a portion of a Vector, which is really a private record type,
you will be able to use the same syntax as if it were an array:

    type Float_Array is array (Positive range <>) of Float;
    
    package My_Vectors is new Container_Vectors( Positive, Float, Float_Array );
    
    use My_Vectors;
    
    V1 : Vector;
    
    V1.Append (1.0);
    
    V1.Append (4.0);
    
    V1.Append (6.0); -- Now internally, we have an array with (1.0, 4.0, 6.0)
    
    V1 (2..3) := (2.0, 3.0); -- Now, without unnecessary copying, we have (1.0, 2.0, 3.0)

The real new trick here is that internally an Element_Array is created and possibly
resized and the range to access is determined by the magic of the range low value
and range high value being respectively placed in the function Reference_Range
as First and Last.  There is no other way to handle a range the way first class
types can be handled as arguments to functions.  Range checking is done inside 
Reference_Range and raises a Constraint_Error if the range is out of the existing
contents of the Vector.  Just like an array, the Vector cannot be appended to with
an assignment to an indexed reference:  "V1 (4..5) := (4.0, 5.0);"  because the range
check would fail.  So the advantages of a private and powerful Vector type remains,
while its usability in place of an array is made more perfect by using a range
to slice a sub-array out of it.  The only remaining difference between a Vector
and an array is that an array can be assigned to initially or with a complete
replacement using the familiar array syntax: "FArray := (1.0, 2.0, 3.0)", while assigning to
a Vector must still be a converted value: "V1 := To_Vector ((1.0, 2.0, 3.0))".



Reference-level explanation
===========================

Variable_Range_Indexing and Constant_Range_Indexing are to operate exactly as
Variable_Indexing and Constant_Indexing, except that they will convert any
range given as an indexing argument to the second and third input arguments of
the associated lookup function.  This allows for the library programmer to
determine how the exact lookup should take place. The programmer may not wish
to match the exact internal first and last indexes of the range with the range
given to the Vector type: 

    V1(1..3)   might not equal  V1.FArray(1..3)
    
    Instead it might mean V1.FArray(3..5)
    
This must be determined by the programmer in the function Reference_Range.
The only requirement for behaving as expected is that both the Vector indexing
range and the internal array indexing use the same type, though either may be different
subtype.

Internally, the featured type, Vector should contain either a constrained array
or an access to an array in the storage pool,
with V1.First and V1.Last components of Index_Type to mark the current slice of the array
that is in use.  The arguments of Reference_Range, First and Last, will be checked
against the cooresponding components of Vector by the Reference_Range function before
attempting to return a Reference_Type.  Then, a declaration renaming a slice of V1.FArray
is made and access to it is returned.  Just as with the access via Variabel_Indexing,
the access type discriminant in Reference_Type is good until the assignment or copying
operation is done.  In preparation for implementaiton, the runtime will have to be checked to make sure it doesn't forget
the renamed array type the access type discriminant points to, though it should not
until the Reference_Type is destroyed.  

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.

Rationale and alternatives
==========================

Right now accessing the inside of a private tagged type is only possible
with Variable_Indexing and Constant_Indexing.  This is good, but it is
unnecessarily verbose to assign to an array slice and the syntax is
so different from that used by an array, that it is not immediately
obvious that the same assignment operation is being performed on a private
type setup like an array and an actual array.  Due to extra functions
being required to acheive the same effect as a call to a slice or an
assignment to a slice of array-like contents, the more code gets dense,
the more it gets difficult to read.  Containers that take the place
of an array, but are more specialized to their particular use should
still be easy and clean to work with.  Making a range effective to use
with tagged type indexing should make code significantly more manageable.

If we do not use utilize ranges in indexing, then we will be forced to
reconsider our methods when using a private tagged type in place of an
array.  Ada is a very consistant and precise language, but due to its
verbosity it is sometimes seen as less than ideal for some projects.
Without compromising Ada's readability, precision, or safety, we can
decrease its verbosity by imlpementing this feature.

Drawbacks
=========




Prior art
=========

This feature does exist in other languages, mostly more abstracted languages,
however Ada has strong typing to control the implementation of slicing an
effectively variable sized array, such that most other languages supporting
this feature do not.  Only (Strict:on) Javascript or Java is really close to matching
this feature with strong typing, as far as I know.  C++ implements Vectors
so similar to arrays that without the dot notation or other subtle differences
you wouldn't be able to distinguish one from an array.  This is the effect
I would consider Ideal in Ada, since the intended use is similar.


Unresolved questions
====================

- I am only concerned about whether the type created renaming part of the
  internal array and possibly with unchecked conversion will remain active
  in the runtime until the access to it is fully finished.  In theory, it
  should, but I am not sure exactly what is going on in the runtime.

Future possibilities
====================


  Another suggestion if anyone wants to formalize it is to add to this idea an aspect
  that finds the indexing range from the right side of the assignment:
  ... with Range_Indexing_By_Assignment => True ...
  V1 := (2.0, 3.0, 4.0) -- assume Positive'First as first and apply as V1 index range
  I haven't thought this idea through, but it is worth noting as the next possible way
  to make a private tagged type behave like an array.  There is no doubt that the 
  private type is not an array, but if it is meant to perform the same general purpose
  as an array with a little extra functionality or safety checks in place, it can only
  help to have it usable with the same syntax as an array.  Unlike the C++ or other
  less precise languages, the private tagged type should not have it's initial value
  set with an implicit conversion from an array.  The initial declaration should always
  be clear about what type a variable is and the initial assignment with a To_Vector()
  call makes this clear.  However, later assignments using To_Vector() are *not* as clear
  and informative as making a range-checked assignment as with the array syntax earlier
  in this paragraph.
  
    
