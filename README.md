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
