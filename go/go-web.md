# web

没 web,不成天下.

---

## 1. Hello world

```go
package main

import (
	"encoding/json"
	"io"
	"log"
	"net/http"
)

type MyBook struct {
	Name    string `json:"name"`
	Author  string `json:"author"`
	Page    int    `json:"page"`
	Summary string `json:"summary"`
}

func ToJSON(universe interface{}) string {
	values, _ := json.Marshal(universe)
	return string(values)
}

// hello world, the web server
func HelloServer(w http.ResponseWriter, req *http.Request) {

	book := MyBook{"Harry Potter", "JK", 400, "About magic"}
	log.Println(ToJSON(book))
	io.WriteString(w, ToJSON(book))
}

func main() {

	http.HandleFunc("/hello", HelloServer)
	err := http.ListenAndServe(":12345", nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}
```

在浏览器访问: `http://127.0.0.1:12345/hello`即可.
