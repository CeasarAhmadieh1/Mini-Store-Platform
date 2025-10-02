---

### `main.go` (core sample code)
```go
package main

import (
	"database/sql"
	"encoding/json"
	"fmt"
	"log"
	"net/http"

	_ "github.com/lib/pq"
)

var db *sql.DB

type Product struct {
	ID          int     `json:"id"`
	Name        string  `json:"name"`
	Description string  `json:"description"`
	Price       float64 `json:"price"`
}

func main() {
	var err error
	connStr := "postgres://postgres:postgres@localhost:5432/store?sslmode=disable"
	db, err = sql.Open("postgres", connStr)
	if err != nil {
		log.Fatal(err)
	}

	http.HandleFunc("/products", productsHandler)
	fmt.Println("Server running on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func productsHandler(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case "GET":
		rows, err := db.Query("SELECT id, name, description, price FROM products")
		if err != nil {
			http.Error(w, err.Error(), 500)
			return
		}
		defer rows.Close()

		var products []Product
		for rows.Next() {
			var p Product
			if err := rows.Scan(&p.ID, &p.Name, &p.Description, &p.Price); err != nil {
				http.Error(w, err.Error(), 500)
				return
			}
			products = append(products, p)
		}
		json.NewEncoder(w).Encode(products)

	case "POST":
		var p Product
		if err := json.NewDecoder(r.Body).Decode(&p); err != nil {
			http.Error(w, err.Error(), 400)
			return
		}
		err := db.QueryRow(
			"INSERT INTO products (name, description, price) VALUES ($1, $2, $3) RETURNING id",
			p.Name, p.Description, p.Price,
		).Scan(&p.ID)
		if err != nil {
			http.Error(w, err.Error(), 500)
			return
		}
		json.NewEncoder(w).Encode(p)

	default:
		http.Error(w, "Method not allowed", 405)
	}
}
