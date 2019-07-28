---
title: Golang Mux
date: 2019-06-08 10:10
tags: golang
---


开始做笔记...
(之前面试被怼了)

<!--more-->

## 路由匹配原理(Regexp in Router)


First, let's go through its *** usage ***:

```go
router:=mux.router()
router.HandlerFunc("/api/v{version}/{category}",handler)
```
```go
//then in handler:
func handler(w http.ResponseWriter,r *http.request){
    vars:=mux.Vars(r)//Get the variables!
    //...
}
```
Let's cut it into two parts:

### HandlerFunc(path,handler)

```golang
// HandleFunc registers a new route with a matcher for the URL path.
// See Route.Path() and Route.HandlerFunc().
func (r *Router) HandleFunc(path string, f func(http.ResponseWriter,
    *http.Request)) *Route {
    return r.NewRoute().Path(path).HandlerFunc(f)
}
```

then let's see  *** Route.Path(path) *** :

```go
// Path -----------------------------------------------------------------------

// Path adds a matcher for the URL path.
// It accepts a template with zero or more URL variables enclosed by {}. The
// template must start with a "/".
// Variables can define an optional regexp pattern to be matched:
//
// - {name} matches anything until the next slash.
//
// - {name:pattern} matches the given regexp pattern.
//
// For example:
//
//     r := mux.NewRouter()
//     r.Path("/products/").Handler(ProductsHandler)
//     r.Path("/products/{key}").Handler(ProductsHandler)
//     r.Path("/articles/{category}/{id:[0-9]+}").
//       Handler(ArticleHandler)
//
// Variable names must be unique in a given route. They can be retrieved
// calling mux.Vars(request).
func (r *Route) Path(tpl string) *Route {//根据传入的url增加正则匹配
	r.err = r.addRegexpMatcher(tpl, regexpTypePath)
	return r
}
```

In *** r.err = r.addRegexpMatcher(tpl, regexpTypePath) ***
the *** regexpTypePath=0 *** means this is for matching Path instead of Host/Port/Prefix/Query

Continue, Go into method *** addRegexpMatcher() ***:

```go
// addRegexpMatcher adds a host or path matcher and builder to a route.
func (r *Route) addRegexpMatcher(tpl string, typ regexpType) error {
	if r.err != nil {
		return r.err
	}
	if typ == regexpTypePath || typ == regexpTypePrefix {
		if len(tpl) > 0 && tpl[0] != '/' {
			return fmt.Errorf("mux: path must start with a slash, got %q", tpl)
		}
		if r.regexp.path != nil {
			tpl = strings.TrimRight(r.regexp.path.template, "/") + tpl
		}
	}
	rr, err := newRouteRegexp(tpl, typ, routeRegexpOptions{// 核心部分
		strictSlash:    r.strictSlash,
		useEncodedPath: r.useEncodedPath,
	})
	if err != nil {
		return err
	}
	for _, q := range r.regexp.queries {
		if err = uniqueVars(rr.varsN, q.varsN); err != nil {
			return err
		}
	}
	if typ == regexpTypeHost {
		if r.regexp.path != nil {
			if err = uniqueVars(rr.varsN, r.regexp.path.varsN); err != nil {
				return err
			}
		}
		r.regexp.host = rr
	} else {
		if r.regexp.host != nil {
			if err = uniqueVars(rr.varsN, r.regexp.host.varsN); err != nil {
				return err
			}
		}
		if typ == regexpTypeQuery {
			r.regexp.queries = append(r.regexp.queries, rr)
		} else {
			r.regexp.path = rr
		}
	}
	r.addMatcher(rr)//这里会把当前matcher加入matcher slice
	return nil
}
```

We will find that we only enter the first *** if *** block (from line 4 - line 11):
Getinto the *** newRouteRegexp() ***

```go
// newRouteRegexp parses a route template and returns a routeRegexp,
// used to match a host, a path or a query string.
//
// It will extract named variables, assemble a regexp to be matched, create
// a "reverse" template to build URLs and compile regexps to validate variable
// values used in URL building.
//
// Previously we accepted only Python-like identifiers for variable
// names ([a-zA-Z_][a-zA-Z0-9_]*), but currently the only restriction is that
// name and pattern can't be empty, and names can't contain a colon.
func newRouteRegexp(tpl string, typ regexpType, options routeRegexpOptions) (*routeRegexp, error) {
    //....
    return ......
}

```

I will roughly split the above code into serveral parts as below:

#### Procedure :

1. 根据括号筛选变量 (braceIndices)
It will check the all *** {variable} *** in Path and save them into a slice:
```go
// braceIndices returns the first level curly brace indices from a string.
// It returns an error in case of unbalanced braces.
func braceIndices(s string) ([]int, error) {
	var level, idx int
	var idxs []int
	for i := 0; i < len(s); i++ {
		switch s[i] {
		case '{':
			if level++; level == 1 {
				idx = i
			}
		case '}':
			if level--; level == 0 {
				idxs = append(idxs, idx, i+1)//注意：这里只是存了variable开始和结尾的index，而不是string
			} else if level < 0 {
				return nil, fmt.Errorf("mux: unbalanced braces in %q", s)
			}
		}
	}
	if level != 0 {
		return nil, fmt.Errorf("mux: unbalanced braces in %q", s)
	}
	return idxs, nil
}
```
====>

2. 移除斜杠 (Remove endSlash)
```go
if has Suffix():
    tpl = tpl[:len(tpl)-1]//remove end slash
```
====>

3. traverse idxs(variables)
``` go

varsN := make([]string, len(idxs)/2)//多少个变量
varsR := make([]*regexp.Regexp, len(idxs)/2)//多少个变量的匹配
pattern := bytes.NewBufferString("")
pattern.WriteByte('^')
reverse := bytes.NewBufferString("")
var end int
var err error
for i := 0; i < len(idxs); i += 2 {//因为之前说过idxs里面是index,+=2实际是遍历下一个变量
    // Set all values we are interested in.
    raw := tpl[end:idxs[i]]//括号之前的string
    end = idxs[i+1]//指的是closed bracket
    parts := strings.SplitN(tpl[idxs[i]+1:end-1], ":", 2)//把变量给弄出来,不过可能有{name:pattern}这种情况，所以要分割开
    name := parts[0]
    patt := defaultPattern //defaultPattern是 [^/]+,即匹配‘/’多次
    if len(parts) == 2 {
        patt = parts[1]//如果有检测到：，就用后面的正则
    }
    // Name or pattern can't be empty.
    if name == "" || patt == "" {
        return nil, fmt.Errorf("mux: missing name or pattern in %q",
            tpl[idxs[i]:end])
    }
    // Build the regexp pattern.
    fmt.Fprintf(pattern, "%s(?P<%s>%s)", regexp.QuoteMeta(raw), varGroupName(i/2), patt)

    // Build the reverse template.
    fmt.Fprintf(reverse, "%s%%s", raw)

    // Append variable name and compiled pattern.
    varsN[i/2] = name//！！！！这个就是变量的存储位置
    varsR[i/2], err = regexp.Compile(fmt.Sprintf("^%s$", patt))
    if err != nil {
        return nil, err
    }
}

```
返回后回到 **func (r *Route) addRegexpMatcher(tpl string, typ regexpType) error** func，
这个函数最后会把match加入matchers里面
```go
// addMatcher adds a matcher to the route.
func (r *Route) addMatcher(m matcher) *Route {
	if r.err == nil {
		r.matchers = append(r.matchers, m)
	}
	return r
}
```



4. 

### Vars()

The place used by mux to save variables called *** context *** (上下文)
And we have to talk about the *** context in 'net/http' ***:
```go
//这个是mux里面的context
// ----------------------------------------------------------------------------
// Context 
// ----------------------------------------------------------------------------

// RouteMatch stores information about a matched route.
type RouteMatch struct {
	Route   *Route
	Handler http.Handler
	Vars    map[string]string

	// MatchErr is set to appropriate matching error
	// It is set to ErrMethodMismatch if there is a mismatch in
	// the request method and route method
	MatchErr error
}

type contextKey int

const (
	varsKey contextKey = iota
	routeKey
)

// Vars returns the route variables for the current request, if any.
func Vars(r *http.Request) map[string]string {
	if rv := contextGet(r, varsKey); rv != nil {//这里会从request的context里面获取key-value pair，然后转换成map
		return rv.(map[string]string)
	}
	return nil
}
```

Then go into *** contextGet(r,varsKey) ***:
```go
//"mux"
func contextGet(r *http.Request, key interface{}) interface{} {
	return r.Context().Value(key)//发现用的是net/http里面的context
}
```
Context struct in *** "net/http" ***可以参考自己写的一个[context](/Context.html)

一切到最后，handleFunc()会调用ServeHTTP（w,req），里面就会把RouteMatch的vars导出，放入http包里的context里
```go
// ServeHTTP dispatches the handler registered in the matched route.
//
// When there is a match, the route variables can be retrieved calling
// mux.Vars(request).
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	if !r.skipClean {
		path := req.URL.Path
		if r.useEncodedPath {
			path = req.URL.EscapedPath()
		}
		// Clean path to canonical form and redirect.
		if p := cleanPath(path); p != path {

			// Added 3 lines (Philip Schlump) - It was dropping the query string and #whatever from query.
			// This matches with fix in go 1.2 r.c. 4 for same problem.  Go Issue:
			// http://code.google.com/p/go/issues/detail?id=5252
			url := *req.URL
			url.Path = p
			p = url.String()

			w.Header().Set("Location", p)
			w.WriteHeader(http.StatusMovedPermanently)
			return
		}
	}
	var match RouteMatch
	var handler http.Handler
	if r.Match(req, &match) {
		handler = match.Handler
		req = setVars(req, match.Vars)//这里，设置vars
		req = setCurrentRoute(req, match.Route)
	}

	if handler == nil && match.MatchErr == ErrMethodMismatch {
		handler = methodNotAllowedHandler()
	}

	if handler == nil {
		handler = http.NotFoundHandler()
	}

	handler.ServeHTTP(w, req)
}
```

