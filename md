# Understanding 'Main.go' 
**Main.go Reference Link: [Click Here](https://github.com/Keen-And-Able/etcd-inventory/blob/sk/main.go)**

### 1. Packaging Program
```
package main
```

This is used to designate the entry point for the compiler to start executing the program from.

### 2. Importation of Packages
```
	import (
    "context"
	"encoding/csv"
	"encoding/json"
	"flag"
	"fmt"
	"log"
	"net/http"
	"os"
	"strings"

	"github.com/tealeg/xlsx"
	clientv3 "go.etcd.io/etcd/client/v3"
    )
```
- **context**: For managing & sending information to different parts of program that handle requests/tasks. It allow setting values of key-value pairs, managing cancellation signals of tasks and setting deadlines for tasks.

- **encoding/csv**: For reading Comma-Separated Values (CSV) file, commonly used for tabular data storage.

- **encoding/json**: For encoding and decoding JSON data.

- **flag**: For handling command-line arguments when running the program from the commmand line. 

- **fmt**: For formatted printing, strings and other values.

- **log**: For printing log messages to flow of error messages through its simple logging interface. 

- **net/http**: For building web applications and interacting with HTTP services.

- **os**: For operating system functionality such as file operations, environment variables & process management through its platform-independent interface.

- **strings**: For manipulating and working with strings.

- **github.com/tealeg/xlsx**: For reading and writing Microsoft Excel (XLSX) files.

- **go.etcd.io/etcd/client/v3**: For interacting with the etcd distributed key-value store.

### 3. Variable Definitions
```
var (
	// File paths
	excelFile = "/home/user/my.db/etcd-inventory/etcd.xlsx"
	csvFile   = "/home/user/my.db/etcd-inventory/myetcd.csv"
	etcdHost  = "localhost:2379"
)
```
Inside the var block, values are defined and related paths are attached.
- **excelFile** = "/home/user/my.db/etcd-inventory/etcd.xlsx"
  
    excelFile defined and file path on system assigned.
- **csvFile** = "/home/user/my.db/etcd-inventory/myetcd.csv"

    csvFile defined and file path on system assigned.

- **etcdHost** = "localhost:2379"

    etcdHost is defined for an etcd server and its host address and ports are assigned.

### 4. 
```
type ServerData map[string]string
```

A type variable can be used to create shorthand representation for a specific type of map with string keys and string values. ServerData map[string]string allows you to create a custom type alias for a map with string keys & string values. 

### 5. 
```
func convertExcelToCSV(excelFile, csvFile string) {
	// Open the Excel file
	xlFile, err := xlsx.OpenFile(excelFile)
	if err != nil {
		log.Fatalf("Failed to open Excel file: %v", err)
	}

	// Create the CSV file
	file, err := os.Create(csvFile)
	if err != nil {
		log.Fatalf("Failed to create CSV file: %v", err)
	}
	defer file.Close()

	// Write data to the CSV file
	writer := csv.NewWriter(file)
	defer writer.Flush()

	// Iterate over sheets and rows in the Excel file
	for _, sheet := range xlFile.Sheets {
		for _, row := range sheet.Rows {
			var rowData []string
			for _, cell := range row.Cells {
				text := cell.String()
				rowData = append(rowData, text)
			}

			// Check if the row is empty
			isEmptyRow := true
			for _, field := range rowData {
				if field != "" {
					isEmptyRow = false
					break
				}
			}

			// Skip empty rows
			if !isEmptyRow {
				writer.Write(rowData)
			}
		}
	}
}
```
- 

    A type variable can be used to create shorthand representation for a specific type of map with string keys and string values. ServerData map[string]string allows you to create a custom type alias for a map with string keys & string values. 

