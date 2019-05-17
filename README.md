# Go2ErrorTree

[Proposal](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md) is not clear for case if we have error tree.

Error-tree created in according to [repository](https://github.com/Konstantin8105/errors).

Error-tree help to grouping the errors.

Description of new language element `errorhandling`:

Go code 
```
errorhandling(et interface(){ Add(error) }){ 
	val1, et = function1()
	val2, et = function2(val1)
	val3, val4, et = function3()
}
```

Internal interpretation `errorhandling`:
```
{ var errLocal error; val1, errLocal = function1(); et.Add(errLocal) }
{ var errLocal error; val2, errLocal = function2(val1); et.Add(errLocal) }
{ var errLocal error; val3, val4, errLocal = function3(); et.Add(errLocal) }
```

Let's suppose on example:
```
func printSum(a, b string) error {
  x, err := strconv.Atoi(a)
  if err != nil {
    return err
  }
  y, err := strconv.Atoi(b)
  if err != nil {
    return err
  }
  fmt.Println("result:", x + y)
  return nil
}
```

Add exist error-tree.

```
func printSum(a, b string) error {
  // create error tree
  et := errors.New("check input data")
  x, err := strconv.Atoi(a)
  if err != nil {
    et.Add(err)
  }
  y, err := strconv.Atoi(b)
  if err != nil {
    et.Add(err)
  }
  // is error-tree have errors
  if et.IsError() {
    return et
  }
  fmt.Println("result:", x + y)
  return nil
}
```

And writing it as:

```go
func printSum(a, b string) error {
  // create error tree
  et := errors.New("check input data")
  errorhandling(et interface{Add(error)}){ // add type just for clearification
    x, et := strconv.Atoi(a)
    y, et := strconv.Atoi(b)
  }
  // is error-tree have errors
  if et.IsError() {
    return et
  }
  fmt.Println("result:", x + y)
  return nil
}
```

Now, we take next example:
```
func CopyFile(src, dst string) error {
	r, err := os.Open(src)
	if err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
	defer r.Close()

	w, err := os.Create(dst)
	if err != nil {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if _, err := io.Copy(w, r); err != nil {
		w.Close()
		os.Remove(dst)
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	if err := w.Close(); err != nil {
		os.Remove(dst)
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}
  
  return nil
}
```

Rewrite in according to error-tree approach:
```
func CopyFile(src, dst string) (err error) {
	// create error tree
	et := errors.New("CopyFile")
	defer func() {
		if et.IsError() {
			er.Add(fmt.Errorf("copy %s %s", src, dst))
		} else {
			err = nil
		}
	}()
	r, err := os.Open(src)
	if err != nil {
		et.Add(err)
		return et
	}
	defer func() {
		et.Add(r.Close())
	}()

	w, err := os.Create(dst)
	if err != nil {
		et.Add(err)
		return et
	}

	if _, err := io.Copy(w, r); err != nil {
		et.Add(err)
		et.Add(w.Close())
		et.Add(os.Remove(dst))
		return et
	}

	if err := w.Close(); err != nil {
		et.Add(err)
		et.Add(os.Remove(dst))
		return et
	}

	return nil
}
```

Rewrite with `errorhandling`:
```go
func CopyFile(src, dst string) (err error) {
	// create error tree
	et := errors.New("CopyFile")
	defer func() {
		if et.IsError() {
			er.Add(fmt.Errorf("copy %s %s", src, dst))
		} else {
			err = nil
		}
	}()
	r, err := os.Open(src)
	if err != nil {
		et.Add(err)
		return et
	}
	defer func() {
		et.Add(r.Close())
	}()

	w, err := os.Create(dst)
	if err != nil {
		et.Add(err)
		return et
	}

	if _, err := io.Copy(w, r); err != nil {
		errorhandling(et) {
			et = err
			et = w.Close()
			et = os.Remove(dst)
		}
		return et
	}

	if err := w.Close(); err != nil {
		errorhandling(et) {
			et = err
			et = os.Remove(dst)
		}
		return et
	}

	return nil
}
```

If we have a tree, then we have to add function for moving by tree as `go.ast.Walk`.

Example of **Walking by error-tree** taked from [git](https://github.com/Konstantin8105/errors):

```golang
type ErrorValue struct {
	ValueName string
	Reason    error
}

func (e ErrorValue) Error() string {
	return fmt.Sprintf("Value `%s`: %v", e.ValueName, e.Reason)
}

func Example() {
	// some input data
	f := math.NaN()
	i := -32
	var s string

	// checking
	var et Tree
	et.Name = "Check input data"
	if math.IsNaN(f) {
		et.Add(ErrorValue{
			ValueName: "f",
			Reason:    fmt.Errorf("is NaN"),
		})
	}
	if f < 0 {
		et.Add(fmt.Errorf("Parameter `f` is negative"))
	}
	if i < 0 {
		et.Add(fmt.Errorf("Parameter `i` is less zero"))
	}
	if s == "" {
		et.Add(fmt.Errorf("Parameter `s` is empty"))
	}

	if et.IsError() {
		fmt.Println(et.Error())
	}

	// walk
	Walk(&et, func(e error) {
		fmt.Fprintf(os.Stdout, "%-25s %v\n", fmt.Sprintf("%T", e), e)
	})

	// Output:
	// Check input data
	// ├──Value `f`: is NaN
	// ├──Parameter `i` is less zero
	// └──Parameter `s` is empty
	//
	// errors.ErrorValue         Value `f`: is NaN
	// *errors.errorString       Parameter `i` is less zero
	// *errors.errorString       Parameter `s` is empty
}
```
