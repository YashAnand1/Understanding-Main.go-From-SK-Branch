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
  
- (A) Opening excel sheet
- (B) Opening CSV
- (C) Converting XLSX to CSV

**(A) Opening excel sheet**
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
  
**(B) Creating CSV**
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
  
**(C) Converting XLSX to CSV**
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
  
- (A) Connecting to etcd
- (B) Opening the CSV
- (C) Parsing the CSV
- (D) Iterating over records & uploading to etcd
- - (D1)  Creating server data dictionary
- - (D2) Setting key-value pairs for data fields
- - (D3) Setting key-value pairs for server data

**(A) Connecting to etcd**
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

**(B) Opening the CSV**
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

**(C) Parsing the CSV**
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

**(D) Iterating Over Records**
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

**(D1) Creating Server Data Dictionary**
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

**(D2) Setting Key-Value Pairs In ETCD For Data Fields**
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

**(D3) Uploading to ETCD**
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
- A variable and etcd key called `etcdKeyData` is created with address using the format `/servers/serverType/serverIP/data`
- A variable and etcd value called `etcdValueData` is created. `json.Marshal` converts `serverData` to JSON format, which can be easier understood by the program.
- If an error occurs in marshalling or converting `serverData`, it is logged and the program continues with the loop.
- Then, the `etcdValueData` value is put into the `etcdKeyData` key and uploaded to etcd. In case an error occurs in uploading server data to etcd, error is logged & printed. 
________________________________
### 7. Getting Server Data
```
func getServerData(w http.ResponseWriter, r *http.Request) { //#7
	// Extract the server type and IP from the URL path
	parts := strings.Split(r.URL.Path, "/")
	serverType := parts[2]
	serverIP := parts[3]

	// Connect to etcd
	etcdClient, err := clientv3.New(clientv3.Config{
		Endpoints: []string{etcdHost},
	})
	if err != nil {
		log.Printf("Failed to connect to etcd: %v", err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}
	defer etcdClient.Close()

	// Construct the etcd key for the server data
	etcdKeyData := fmt.Sprintf("/servers/%s/%s/data", serverType, serverIP)

	// Get the server data from etcd
	response, err := etcdClient.Get(context.Background(), etcdKeyData)
	if err != nil {
		log.Printf("Failed to get server data: %v", err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}

	// Check if the key exists
	if len(response.Kvs) == 0 {
		http.Error(w, "Server data not found", http.StatusNotFound)
		return
	}

	// Extract the server data value
	var serverData ServerData
	err = json.Unmarshal(response.Kvs[0].Value, &serverData)
	if err != nil {
		log.Printf("Failed to unmarshal server data: %v", err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}

	// Extract the specific field value from the server data
	field := parts[4]
	value, ok := serverData[field]
	if !ok {
		http.Error(w, "Field not found", http.StatusNotFound)
		return
	}

	// Write the field value as the response
	w.Header().Set("Content-Type", "text/plain")
	w.Write([]byte(value))
}
```
This code can be understood better by breaking it into 4 different parts, which we will be discussing later:
  
- (A) Extrcting server IP & Type From URL
- (b) Connect to etcd
- (C) Creating ETCD Key for server data
- (D) Getting server data from etcd
- (E) Ensuring the key's existence
- (F) Extracting server data value
- (G) Extracting **SPECIFIC** server data value
- (H) Writing field value as the response

**(A) Extrcting server IP & Type**
```
func getServerData(w http.ResponseWriter, r *http.Request) { //#7
	// Extract the server type and IP from the URL path
	parts := strings.Split(r.URL.Path, "/")
	serverType := parts[2]
	serverIP := parts[3]
```
- `func getServerData` is created to help get stored information related to servers.
- This function has 2 parameters/information that it expects to be provided when called: `w http.ResponseWriter, r *http.Request`.
- - `w` represents `http.ResponseWriter` which is a tool that allows communicating back to client that made HTTP request.
- - `r` represents a pointer to `*http.Request` which points to the original location of `http.request` every time the function is called, instead of making copies.
- `parts := strings.Split(r.URL.Path, "/")` divides the `r.URL.Path` like server/192.10.90.1/data after every `/`. The result which is an array of divided parts are addded to variable `parts`.
-  `serverType := parts[2]` & `serverIP := parts[3]` extracts the server type and IP from the divided URL path. Parts 2nd & 3rd variable are extracted for 3rd & 4th element. 

**(B) Extrcting server IP & Type**
```
	// Connect to etcd
	etcdClient, err := clientv3.New(clientv3.Config{
		Endpoints: []string{etcdHost},
	})
	if err != nil {
		log.Printf("Failed to connect to etcd: %v", err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}
	defer etcdClient.Close()
```
- The `etcdClient` variable creates a new etcd connection with new configurations. 
- The connection is to be made between `etcdClient` and `etcdHost`. 
- `Endpoints: []string{etcdHost}` contains the address of etcdHost or the end connection, allowing the connection to take place.
- If the connection cannot be established due to an error, logging of error is done. Similarly, `http.Error(w, "Internal Server Error", http.StatusInternalServerError)` sends an HTTP response error to the client.
- When the connection between `etcdHost` & `etcdClient` is no longer required, it is to be closed later using `defer etcdClient.Close()`.

**(C) Creating ETCD Key for server data**
```
	// Construct the etcd key for the server data
	etcdKeyData := fmt.Sprintf("/servers/%s/%s/data", serverType, serverIP)
```
- `etcdKeyData` variable is created with '/servers/serverType/serverIP/data' as its assigned value.
- This way a complete key inclusive of serverType & ServerIP is created  for the server data. `fmt.Sprintf` allows taking in actual value of serverType & serverIP from the URL path.

**(D) Getting server data from etcd**
```
	// Get the server data from etcd
	response, err := etcdClient.Get(context.Background(), etcdKeyData)
	if err != nil {
		log.Printf("Failed to get server data: %v", err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}
```
- `response` variable is create to hold in the value obtained from the `etcdKeyData` key. This key specifies the ServerIP & Type and retrieves the required data.
- If `err` error occurs during the process of getting the data associated with the `etcdKeyData` key, it is logged.v
- Using `http.Error`, a response is sent to client with an error message and HTTP error code.

**(E) Ensuring the key's existence**
```
	// Check if the key exists
	if len(response.Kvs) == 0 {
		http.Error(w, "Server data not found", http.StatusNotFound)
		return
	}
```
- If the length of key-values or kvs in the `response` are equal to 0, then that means that the key is empty.
- Using `http.Error`, a response is sent to client with an error message and HTTP error code to let the user know that "server data could not be found".

**(F) Extracting server data value**
```
	// Extract the server data value
	var serverData ServerData
	err = json.Unmarshal(response.Kvs[0].Value, &serverData)
	if err != nil {
		log.Printf("Failed to unmarshal server data: %v", err)
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}
```
- `ServerData` variable is created and it is specified that it will be holding ServerData (which contains keys & values).
- The decoding of the `response.Kvs[0].Value` is done using `json.Unmarshal` and the decoded data is stored in `serverData`.
- If an error occurs due to which the decoding could not be done, `log.Printf` is used to log the error.
- Similarly using `http.Error`, a response is sent to client with an error message and HTTP error code to let the user know that there was an "Internal Server Data". w = http.ResponseWriter

**(G) Extracting <u>SPECIFIC</u> server data value**
```
	// Extract the specific field value from the server data
	field := parts[4]
	value, ok := serverData[field]
	if !ok {
		http.Error(w, "Field not found", http.StatusNotFound)
		return
	}
```
- The `field` variable is used to assign `parts[4]` or the 4th element of the URL path to it.
- Using the `field` variable as the key, the value associated with it in the `serverData` is extracted.
- The extracted value of the `field` key is stored inside the `value` variable.
- `ok` variable signifies if value was found or not. If not, using `http.Error`, a response is sent to client with an error message and HTTP error code to let the user know that the Field could not be found.

**(H) Writing field value as the response**
```
	// Write the field value as the response
	w.Header().Set("Content-Type", "text/plain")
	w.Write([]byte(value))
}
```
- `w.Header().Set` designates the header to have the content type of plain text. This is to ensure that the data displayed to client will be readable.
- In order to write the value of the specfic key, `w.Write([]byte(value))` is used to convert the value into bytes and have it displayed as plain text.
___
### 8. Main Program
```
func main() { //#8
	// Convert Excel to CSV
	convertExcelToCSV(excelFile, csvFile)
	log.Println("Excel file converted to CSV successfully.")

	// Parse command-line flags
	flag.Parse()

	// Upload CSV data to etcd
	uploadToEtcd()

	// Start API server
	log.Println("Starting API server...")
	http.HandleFunc("/servers/", getServerData)
	err := http.ListenAndServe(":8080", nil)
	if err != nil {
		log.Fatalf("Failed to start API server: %v", err)
	}
}
```
The functions which have been defined above are called here in the main function. This function is responsible for the following:
- (A) Converting Excel To CSV
- (B) Parsing Command Line Flags
- (C) Uploading CSV Data To ETCD
- (D) Starting API erver

**(A) Converting Excel To CSV**
```
func main() { //#8
	// Convert Excel to CSV
	convertExcelToCSV(excelFile, csvFile)
	log.Println("Excel file converted to CSV successfully.")
```
- `convertExcelToCSV` function is called with the file paths of the excel & CSV file as the arugements. Then the success message is logged.

**(B) Parsing Command Line Flags**
```
	// Parse command-line flags
	flag.Parse()
```
- `Flag.Parse()` allows extracting and reading the command line flags used when running the code. 

**(C) Uploading CSV Data To ETCD**
```
	// Upload CSV data to etcd
	uploadToEtcd()
```
- The `uploadToEtcd()` function is called to start uploading the CSV data to the etcd server.

**(D) Starting API erver**
```
	// Start API server
	log.Println("Starting API server...")
	http.HandleFunc("/servers/", getServerData)
	err := http.ListenAndServe(":8080", nil)
	if err != nil {
		log.Fatalf("Failed to start API server: %v", err)
	}
}
```
- ``
