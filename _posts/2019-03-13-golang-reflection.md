---
layout: post
title: "Reflection with Golang"
description: "From Wikipedia : 'In computer science, reflection is the ability of a computer program to examine, introspect, and modify its own structure and behavior at runtime.' Golang has a reflection package named reflect but you can hardly find examples of its use whether in tutorials or even in large projects."
tags: [Golang,Reflection,Programming]
---

From Wikipedia : "In computer science, reflection is the ability of a computer program to examine, introspect, and modify its own structure and behavior at runtime."
Golang has a reflection package named reflect but you can hardly find examples of its use whether in tutorials or even in large projects.

For instance, Kubernetes code mainly uses the deep equality ability from this package and a few type information gathering. But what about modifying structure and behavior ?

There's a case where this could be very handy. Have you ever been in the situation where you're coding a software that resorts heavily on channels ?
Actually, if you're fully using Go, this should happen often. But what if many of the functions you're calling are just returning plain old types aka non channel types ?
Transforming the original function to adapt it to you need is clearly a use case of reflection. How to do it with Go reflection ?
See below the code and comments to explain the different attempts to achieve that in the most natural way i.e. without reflection code.

{% highlight golang %}

// main
package main

import (
	"fmt"
	"reflect"
)

// boring function for the sake of test.
func foo(a, b, q int) (int, int, error) {
	return b * q, a + b*q, nil
}

/*
make_function receives a function as argument and returns an other function as a Value.
This new function encapsulates the original one and returns channel for every original returned value.
*/
func make_function(f interface{}) reflect.Value {
	v := reflect.ValueOf(f)
	t := reflect.TypeOf(f)

/* 
fn is the function that is used to encapsulate the original function
The signature of this function is imposed by the MakeFunc call below. 
reflect.Value slices used as arguments and returned values types make fn suitable to replace any function.
*/

	fn := func(args []reflect.Value) []reflect.Value {
		// results contains every channel ...
		results := make([]reflect.Value, t.NumOut())
		// ... corresponding to each returned value type.
		for i := 0; i < t.NumOut(); i++ {
			/* 
			Every channel is built by reflection. 
			ChanOf creates the type of the channel from the type of the corresponding returned value.
			MakeChan builds the channel from the specification returned by ChanOf.
			 Pay attention that MakeChan can only accept channel that sends and receives (BothDir).
			 */
			results[i] = reflect.MakeChan(reflect.ChanOf(reflect.BothDir, t.Out(i)), 0)
		}
		// The original function is called.
		values := v.Call(args)
		// The returned values are now dispatched to ...
		for i, value := range values {
			// ... corresponding channels
			// The sending is done through goroutines to prevent deadlock.
			go func(k int, val reflect.Value) {
				results[k].Send(val)
			}(i, value)
		}

		return results
	}

	// ins ans outs are slices of types respectively of arguments and returned values.
	ins, outs := make([]reflect.Type, t.NumIn()), make([]reflect.Type, t.NumOut())
	// ins conserves the original types of arguments ...
	for i := 0; i < t.NumIn(); i++ {
		ins[i] = t.In(i)
	}
	// ... but outs replaces every returned value by its channel counterpart.
	for i := 0; i < t.NumOut(); i++ {
		outs[i] = reflect.ChanOf(reflect.BothDir, t.Out(i))
	}
	// reflect.FuncOf defines the type of the new function.
	// Then reflect.MakeFunc implements dynamically the new function from the definition
	return reflect.MakeFunc(reflect.FuncOf(ins, outs, false), fn)
}

func main() {

	// I get the new function from the original one.
	newF := make_function(foo)
	// As a the new function is a Value, it's called through the reflection API.
	values := newF.Call([]reflect.Value{reflect.ValueOf(10), reflect.ValueOf(34), reflect.ValueOf(5)})
	/* 
The channels are returned as slice of Value, so you can't use them directly in a select statement.You must resort to reflection API to build dynamically the select case to get the returned values.First, you create the select cases, not as statements but as objects.
	*/
	prod := reflect.SelectCase{Dir: reflect.SelectRecv, Chan: values[0]}
	prodsum := reflect.SelectCase{Dir: reflect.SelectRecv, Chan: values[1]}
	errSel := reflect.SelectCase{Dir: reflect.SelectRecv, Chan: values[2]}
	var (
		prodValue, proSumValue int
		err                    error
	)
	for i := 0; i < 3; i++ {
	  // Implements the select as an object containing the select cases built above.
		chosen, recv, _ := reflect.Select([]reflect.SelectCase{prod, prodsum, errSel})
		value := recv.Interface()
		switch chosen {
		case 0:
			prodValue = value.(int)
		case 1:
			proSumValue = value.(int)
		case 2:
			if value != nil {
				err = value.(error)
			}
		}

	}
	fmt.Println(prodValue, proSumValue, err)

}

{% endhighlight %}

Not bad, the target function is encapsulated and every of its returned value is now transferred to a channel.
But wait a minute, the code is really cumbersome and verbose,
Maybe it could be more effective and readable if I could pass my own channels. 

{% highlight golang %}
// main
package main

import (
	"fmt"
	"log"
	"reflect"
)

func foo(a, b, q int) (int, int, error) {
	return b * q, a + b*q, nil
}

// Now I pass a variadic arg of chans of reflect.Value.
func make_function_chan(f interface{}, chans ...chan reflect.Value) reflect.Value {
	v := reflect.ValueOf(f)
	t := reflect.TypeOf(f)
	fn := func(args []reflect.Value) []reflect.Value {

		values := v.Call(args)
		for i, value := range values {

			go func(k int, val reflect.Value) {
				c := chans[k]
				// Channel transmitted ? Ok, otherwise skip it.
				if c != nil {
					c <- val
				}

			}(i, value)
		}
		// The encapsulating function is now of type 'void' as all values will be transmitted by channels.
		return []reflect.Value{}
	}

	ins, outs := make([]reflect.Type, t.NumIn()), []reflect.Type{}
	for i := 0; i < t.NumIn(); i++ {
		ins[i] = t.In(i)
	}
	return reflect.MakeFunc(reflect.FuncOf(ins, outs, false), fn)
}

func main() {
	r1, r2, errorChan := make(chan reflect.Value), make(chan reflect.Value), make(chan reflect.Value)
	newF := make_function_chan(foo, r1, r2, errorChan)
	newF.Call([]reflect.Value{reflect.ValueOf(10), reflect.ValueOf(34), reflect.ValueOf(5)})
	// Thanks to using plain old channels, I can write plain old select statement.
	for i := 0; i < 3; i++ {
		select {
		case a := <-r1:
			fmt.Println(a.Interface().(int))
		case b := <-r2:
			fmt.Println(b.Interface().(int))
		case errValue := <-errorChan:
			err := errValue.Interface()
			if err != nil {
				log.Fatal(err.(error))
			}
		}
	}
}

{% endhighlight %}

Much better ! But still I'm forced to provide as many channels as returned values. But what if I'm not concerned with every value ? 
I should be able to distinguish between directly returned values and 'channeled' returned values. 
In addition, calling the function is still very odd and involves reflection API. Select statement is still cribbled with reflection API as well.
Can we clean the code to make all the reflection plumbing transparent?

{% highlight golang %}

// main
package main

import (
	"fmt"
	"reflect"
)

func add(a, b, q int) (int, int, error) {
	return b * q, a + b*q, nil
}

type WrapFunc func(args ...interface{}) []interface{}

func make_function_mix_wrap(f interface{}, chans ...chan interface{}) WrapFunc {
	v := reflect.ValueOf(f)
	t := reflect.TypeOf(f)
	// outs will be set later.
	ins, outs := make([]reflect.Type, t.NumIn()), []reflect.Type{}
	for i := 0; i < t.NumIn(); i++ {
		ins[i] = t.In(i)
	}

	var j int
/*
Now I'm only retaining value type of returned values if no channel was provided for the corresponding returned value.	
 I must keep the matching of returned values from the orginal function and returned values from encapsulating function.
*/
	outMatchings := make(map[int]int)
	for i, ch := range chans {
		if ch == nil {
			// I keep only returned values types for returned values without corresponding channel.
			outs = append(outs, t.Out(i))
			outMatchings[i] = j
			j++
		}

	}

	fn := func(args []reflect.Value) []reflect.Value {

		results := make([]reflect.Value, j)
		values := v.Call(args)
		for i, value := range values {
			c := chans[i]
			// If no channel for this returned value, I add the result to the returned values of encapsulating function.
			// I need there the matching as stated above.
			if c == nil {
				results[outMatchings[i]] = value
			} else {
				go func(valueChan chan interface{}, val reflect.Value) {
					valueChan <- val.Interface()
				}(c, value)
			}
		}
		return results
	}
	// Now the first encapsulating function ...
	newF := reflect.MakeFunc(reflect.FuncOf(ins, outs, false), fn)
	// ... is itself encapsulated for the sake of consistency with casual function calls.
	return func(args ...interface{}) []interface{} {
		newArgs := make([]reflect.Value, len(args))
		for i, v := range args {
			newArgs[i] = reflect.ValueOf(v)
		}
		encapsulatingValues := newF.Call(newArgs)
		returnedValues := make([]interface{}, len(encapsulatingValues))
		for i, v := range encapsulatingValues {
			returnedValues[i] = v.Interface()
		}
		return returnedValues

	}

}
func main() {
	prodChan := make(chan interface{})
	newF := make_function_mix_wrap(add, prodChan, nil, nil)
	values := newF(10, 34, 5)
	for _, value := range values {
		fmt.Println(value)
	}
	prod := <-prodChan
	fmt.Println("prod", prod)
}

{% endhighlight %}

Finally, something usable even if it lacks lot of guards.It could even be changed to return new channels with results instead of getting one from the caller.
I leave it to the reader as a useful exercice.
This little experience shows clealy that reflection in Golang can be really powerful. 
It's really underestimated but to my eyes it should gain in easiness. Encapsulating and transforming a function also known as Decorator is heavy to implement.
Compare it to Python,Javascript or even to languages like Erlang/Elixir with their very versatile macros !
Nevertheless, it can make your code more usable and readable and reflection API opens new ways to improve your code. 
And now you're expert in a very advanced language feature !