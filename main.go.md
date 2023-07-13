# Understanding 'Main.go' 
**Main.go Reference Link: [Click Here](https://github.com/Keen-And-Able/etcd-inventory/blob/sk/main.go)** | * THIS DOCUMENT IS STILL IN PROGRESS *
_________________
### 1. Packaging Program
```
package main
```
- The goal of starting the program with this is to designate the entry point for the compiler to start executing the program from.
- It also allows using the various functions and packages that are to be used in the program. 
________________
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
- **context**: 
For managing & sending information to different parts of program that handle requests/tasks. It allow setting values of key-value pairs, managing cancellation signals of tasks and setting deadlines for tasks.

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
________________
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

- **excelFile = "/home/user/my.db/etcd-inventory/etcd.xlsx":**
excelFile defined and file path on system assigned.
- **csvFile = "/home/user/my.db/etcd-inventory/myetcd.csv"**:
csvFile defined and file path on system assigned.
- **etcdHost = "localhost:2379"**:
etcdHost is defined for an etcd server and its host address and ports are assigned.
_______________
### 4. Server Data
```
type ServerData map[string]string
```

- This data type allows storing important information related to servers for interacting with them.
- Helps manage various server information by keeping track of every server and what it does.
__________________
### 5. Excel To CSV Conversion
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
This code can be understood better by breaking it into 3 different parts, which we will be discussing later:
  
- Opening excel sheet
- Opening CSV
- Converting XLSX to CSV

**a. Opening excel sheet**
```
func convertExcelToCSV(excelFile, csvFile string) {
	// Open the Excel file
	xlFile, err := xlsx.OpenFile(excelFile)
	if err != nil {
		log.Fatalf("Failed to open Excel file: %v", err)
	}
```
- Data from excel is converted to CSV format inside the `ConvertExcelToCSV function`. 
- The excel file opened from its defined filepath by mentioning the excelFile variable
- In case the file is not found or cannot be opened due to an error, the error is logged & program stops.
  
**b. Creating CSV**
```
// Create the CSV file
	file, err := os.Create(csvFile)
	if err != nil {
		log.Fatalf("Failed to create CSV file: %v", err)
	}
	defer file.Close()
```
- A CSV file is created and stored in the defined file path.
- The CSV file is created and if due to an error, it cannot be created, the error is logged and program stops.
- Additionally, it is ensured that the created CSV file would close after it is no longer being used.
  
**c. Converting XLSX to CSV**
```
// 	Write data to the CSV file
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
- csv.NewWriter does the conversion of XLSX data to CSV. 
- It looks through each Excel sheet and row and collects the row data and stores it to rowData list.
- Additionally, if any cell is found to be empty then that cell is skipped.
____________________________________________________________________________

### 6. Uploading To ETCD
```
func uploadToEtcd() { //#6
	// Connect to etcd
	etcdClient, err := clientv3.New(clientv3.Config{
		Endpoints: []string{etcdHost},
	})
	if err != nil {
		log.Fatalf("Failed to connect to etcd: %v", err)
	}
	defer etcdClient.Close()

	// Read the CSV file
	file, err := os.Open(csvFile)
	if err != nil {
		log.Fatalf("Failed to open CSV file: %v", err)
	}
	defer file.Close()

	// Parse the CSV file
	reader := csv.NewReader(file)
	records, err := reader.ReadAll()
	if err != nil {
		log.Fatalf("Failed to read CSV file: %v", err)
	}

	// Iterate over the records and upload to etcd
	headers := records[0]
	for _, record := range records[1:] {
		serverIP := record[0]
		serverType := record[1]
		serverData := make(ServerData)

		// Create server data dictionary
		for i := 2; i < len(headers); i++ {
			header := headers[i]
			value := record[i]
			serverData[header] = value
		}

		// Set key-value pairs in etcd for each data field
		for header, value := range serverData {
			etcdKey := fmt.Sprintf("/servers/%s/%s/%s", serverType, serverIP, header)
			etcdValue := value
			_, err := etcdClient.Put(context.Background(), etcdKey, etcdValue)
			if err != nil {
				log.Printf("Failed to upload key-value to etcd: %v", err)
			}
		}

		// Set key-value pair for server data
		etcdKeyData := fmt.Sprintf("/servers/%s/%s/data", serverType, serverIP)
		etcdValueData, err := json.Marshal(serverData)
		if err != nil {
			log.Printf("Failed to marshal server data: %v", err)
			continue
		}
		}
	}

	log.Println("Server details added to etcd successfully.")
}
```
This code can be understood better by breaking it into 4 different parts, which we will be discussing later:
  
- Connecting to etcd
- Opening the CSV
- Parsing the CSV
- Iterating over records & uploading to etcd
- - Creating server data dictionary
- - Setting key-value pairs for data fields
- - Setting key-value pairs for server data

**a. Connecting to etcd**
```
func uploadToEtcd() { //#6
	// Connect to etcd
	etcdClient, err := clientv3.New(clientv3.Config{
		Endpoints: []string{etcdHost},
	})
	if err != nil {
		log.Fatalf("Failed to connect to etcd: %v", err)
	}
	defer etcdClient.Close()
```
- A function called `uploadToEtcd` contains `etcdClient` variable.
- The `etcdClient` variable creates a new etcd connection with new configurations. 
- The connection is to be made between `etcdClient` and `etcdHost`. 
- `Endpoints: []string{etcdHost}` contains the address of etcdHost or the end connection, allowing the connection to take place.
- If the connection cannot be established due to an error, logging of error is done and program closed. 
- Similarly, when the connection between `etcdHost` & `etcdClient` is no longer required, it is to be closed later using `defer etcdClient.Close()`.

**b. Opening the CSV**
```	
	// Read the CSV file
	file, err := os.Open(csvFile)
	if err != nil {
		log.Fatalf("Failed to open CSV file: %v", err)
	}
	defer file.Close()
```
- `os.Open(csvFile)` is used to open the CSV file from it's designated file path.
- In case the CSV file cannot be opened due to a error, then the error is logged and code exits.
- Lastly, `defer file.Close()` is used for closing the opened CSV file that was opened, once the work is done.

**c. Parsing the CSV**
```	
	// Parse the CSV file
	reader := csv.NewReader(file)
	records, err := reader.ReadAll()
	if err != nil {
		log.Fatalf("Failed to read CSV file: %v", err)
	}
```
- `reader := csv.NewReader(file)` assigns the previously opened CSV file to 'reader' variable so that content of the file can be processed & read.
- `records, err := reader.ReadAll()` allows reading all the records or rows at once from the CSV file. If unable to read CSV due to error, logging of error is done & code exited.

**d. Iterating Over Records**
```	
	// Iterate over the records and upload to etcd
	headers := records[0]
	for _, record := range records[1:] {
		serverIP := record[0]
		serverType := record[1]
		serverData := make(ServerData)
```
- The first row or records which represents the column name is assigned to the `headers` variable. This row includes names for the columns. 
- Starting from row 2, a loop is run to record each record to assign it to `record` variable. 
- Starting from second row, a loop runs through every row.
- `serverIP` is assigned to the first column and `serverType` to the second.
- A `serverData` variable for storing server data is created. 

**d.1. Creating Server Data Dictionary**
```	
	// Create server data dictionary
	for i := 2; i < len(headers); i++ {
	header := headers[i]
	value := record[i]
	serverData[header] = value
	}
```
- A loop is started from the third column as first 2 are already in use by `serverIP` & `serverData`. It continues until columns total columns/headers are reached.
- It looks through column names and assigns them to variable `header`. The values inside these columns which are stored as rows are assigned to `value` variable.
-  Using `header` as label, `value` is added to the `serverData`. This process repeats until all the headers have been used.

**d.2. Setting Key-Value Pairs In ETCD For Data Fields**
```	
	// Set key-value pairs in etcd for each data field
	for header, value := range serverData {
		etcdKey := fmt.Sprintf("/servers/%s/%s/%s", serverType, serverIP, header)
		etcdValue := value
		_, err := etcdClient.Put(context.Background(), etcdKey, etcdValue)
		if err != nil {
		log.Printf("Failed to upload key-value to etcd: %v", err)
		}
	}
```
- A loop is run to examine each key-value pair of headers & values in the `serverData`.
- In `etcdKey := fmt.Sprintf("/servers/%s/%s/%s", serverType, serverIP, header)`, data from the the `serverData` is set per iteration. Key is made for each header.
- In `etcdValue := value`, every header's value is set per iteration. 
- `etcdClient.Put` is used to have the key-value pairs entered into the etcd. If an error occurs, it is logged with an error message.

**d.3. Setting Key-Value Pairs In ETCD For Data Fields**
```	
	// Set key-value pair for server data
	etcdKeyData := fmt.Sprintf("/servers/%s/%s/data", serverType, serverIP)
	etcdValueData, err := json.Marshal(serverData)
	if err != nil {
		log.Printf("Failed to marshal server data: %v", err)
		continue
	}
	_, err = etcdClient.Put(context.Background(), etcdKeyData, string(etcdValueData))
	if err != nil {
		log.Printf("Failed to upload server data to etcd: %v", err)
		}
	}
```
- A variable called `etcdKeyData` is assigned to store an address/location of 
  
