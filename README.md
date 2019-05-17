# Go2ErrorTree

[Proposal](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md) is not clear for case if we have error tree.

Error-tree created in according to [repository](https://github.com/Konstantin8105/errors).

Let's suppose on example:
```go
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

```go
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
```go
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
```go
func CopyFile(src, dst string) error {
	// create error tree
	et := errors.New("CopyFile")
	defer func() {
		if et.IsError() {
			er.Add(fmt.Errorf("copy %s %s: %v", src, dst, err))
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
		er.Add(err)
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
