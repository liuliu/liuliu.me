---
date: '2010-03-18 00:30:24'
layout: post
slug: lets-abusively-use-cpp-macros
status: publish
title: Let's abusively use cpp macros
wordpress_id: '810'
categories:
- Eyes
tags:
- code optimization
- framework
---

Since I started the ccv project (http://github.com/liuliu/ccv) a few weeks ago, one motivation is to write a pure-c (C99 standard) library for computer vision. The trouble is, for a computer vision library, we deal with 4 types of matrix, unsigned char, integer, single precision, double precision. BLAS has separate functions to do calculation on the 4 types. But I found it is easier for user to just have one matrix structure with a flag to label types.

It is convenient unless you choose to play around with raw data (as the library writer did). And it did impose a problem to library writer, do I need to maintain 4 copies of code just with slight change of data type? That is one of the few times you miss the good part of C++, I mean, the template. However, with very abusive usage of macro, you can still get the candies off the shelf with the price of more compilation time.

The classic education of code optimization told everyone to put if branch outside of a deep for-loop, and everyone takes it as a common sense. However, behind the scene, compiler does a lot in such scenario and it does well. Let me show you an example (ccv.h file can be found [here](http://github.com/liuliu/ccv/blob/master/src/ccv.h)):

    
    #include "ccv.h"
    
    void copy_data_to_float(ccv_dense_matrix_t* x, float* out)
    {
    	float* out_ptr = out;
    	unsigned char* m_ptr = x->data.ptr;
    	int i, j;
    	for (i = 0; i < x->rows; i++)
    	{
    		for (j = 0; j < x->cols; j++)
    			out_ptr[j] = ccv_get_value(x->type, m_ptr, i);
    		out_ptr += x->cols;
    		m_ptr += x->step;
    	}
    }
    
    int main(int argc, char** argv)
    {
    	ccv_dense_matrix_t* x = ccv_dense_matrix_new(100, 100, CCV_8U | CCV_C1, NULL, NULL);
    	float* out = (float*)malloc(sizeof(float) * 100 * 100);
    	copy_data_to_float(x, out);
    	free(out);
    	ccv_matrix_free(x);
    	ccv_gabarge_collect();
    	return 0;
    }


When compile the code with gcc and -O3 flag, you can get some nice assembly that put the inner switch statement (hided by the ccv_get_value macro) outside the for loop. More officially, here we use the -funswitch-loop flag to optimize performance. It is a trick, but gives C some ability that template has (and cons: generating larger binary). In the end of the day, we do have a clearer C code with reasonable performance.

If the compiler is so clever every time, the world will be free of wars, hunger and disease. The problem is, you can never be certain that compiler will do the smart thing. For example, if in a case you have several switches, most of the time, compiler will fail to unswitch them. It is just too much mutations (even for no-nested switches, when you unswitch them, the mutations should be first switch cases * second switch cases * ... * n-th switch cases, in other word, exponential). Even that is the case, we do want to unswitch them all (4 switches, and each switch has 4 cases will result 256 different for-loops, still manageable on modern computers). How to do that semi-automatically?

First thought is to define these for-loops as macro, and somehow expanded them. It should look like some kind of:

    
    #define get_value(x, i) (int*)(x)[i]
    #define for_block \
    	for (i = 0; i < 100; i++) \
    		y[i] = get_value(x, i);


If only we can define different get_value for different x types, the for_block can be expanded. One way to do it is to use .def file. However, #include instruction cannot be implemented inside a macro, that means we cannot wrap our solution into one nice macro.

Another thought comes to mind is: instead of define a macro first, just pass the macro as parameter. The following code did this:

    
    #define for_block(for_get) \
    	for (i = 0; i < 100; i++) \
    		y[i] = for_get(x, i);
    #define get_int_value(x, i) ((int*)(x))[i]
    #define get_float_value(x, i) ((float*)(x))[i]
    switch (type)
    {
    	case INT:
    		for_block(get_int_value);
    		break;
    	case FLOAT:
    		for_block(get_float_value);
    		break;
    }


It is very close to the final goal, I can even imagine a nice wrapper around the method:

    
    #define getter(type, block) switch (type) { \
    	case INT: \
    		block(get_int_value); \
    		break; \
    	case FLOAT: \
    		block(get_float_value); \
    		break; \
    }
    getter(type, for_block);


We can write out our getter, the setter for our matrix operation for loop immediately, hurrah. Except one thing, you cannot use them conjunctively. Then, what's the point of having one switch out? I want do something like set(y, i, get(x, i)), and how?

A nice feature in C99 standard is the support of variadic macros. Variadic macro, just like the name suggests, can take variable length of arguments. It is a good start point to write our joinable getter and setter. The code is:

    
    #define getter(type, block, rest...) switch (type) { \
    	case INT: \
    		block(rest, get_int_value); \
    		break; \
    	case FLOAT: \
    		block(rest, get_float_value); \
    		break; \
    }
    getter(type, for_block);


The rest part makes all different, now you can do something like: setter(type_a, getter, type_b, for_block); and the for_block will take two arguments: (set, get) in the same order you call it. The thought process is left for readers, but there is one pitfall: for one getter case, the for_block will take the first argument as dummy (because of the rest argument inserted before the real get macro).

For multiple getter, it is trickier. Because all macros are not recursive, you have to define several getters that has the exact same code but different name in order to use multiple getter. And yes, like all the macros, it is hard to debug (in this case, worse, since one mistake can generate tens or hundreds compile-time errors).

Finally, I will show you some code in action (for pairwise addition of matrix a, b to c):

    
    #include "ccv.h"
    unsigned char* a_ptr = a->data.ptr;
    unsigned char* b_ptr = b->data.ptr;
    unsigned char* c_ptr = c->data.ptr;
    #define for_block(__for_get_a, __for_get_b, __for_set) \
    for (i = 0; i < a->rows; i++) \
    { \
    	for (j = 0; j < a->cols; j++) \
    	{ \
    		__for_set(c_ptr, j, __for_get_a(a_ptr, j) + __for_get_b(b_ptr, j)); \
    	} \
    	a_ptr += a->step; \
    	b_ptr += b->step; \
    	c_ptr += c->step; \
    }
    ccv_matrix_getter_a(a->type, ccv_matrix_getter_b, b->type, ccv_matrix_setter, c->type, for_block);
    #undef for_block


If you want to use plain old C way and unswitch for-loop, you have to write 64 for-loops to cover all the cases.
