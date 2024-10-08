# Uploads and downloads {#action-transfer}



Transferring files to and from the user is a common feature of apps.
You can use it to upload data for analysis, or download the results as a dataset or as a report.
This chapter shows the UI and server components that you'll need to transfer files in and out of your app.


```r
library(shiny)
```

## Upload {#upload}

We'll start by discussing file uploads, showing you the basic UI and server components, and then showing how they fit together in a simple app.

### UI

The UI needed to support file uploads is simple: just add `fileInput()` to your UI.


```r
ui <- fluidPage(
  fileInput("upload", "Upload a file")
)
```

Like most other UI components, there are only two required arguments: `id` and `label`.
The `width`, `buttonLabel` and `placeholder` arguments allow you to tweak the appearance in other ways.
I won't discuss them here, but you can read more about them in `?fileInput`.

### Server

Handling `fileInput()` on the server is a little more complicated than other inputs.
Most inputs return simple vectors, but `fileInput()` returns a data frame with four columns:

-   `name`: the original file name on the user's computer.

-   `size`: the file size, in bytes.
    By default, the user can only upload files up to 5 MB.
    You can increase this limit by setting the `shiny.maxRequestSize` option prior to starting Shiny.
    For example, to allow up to 10 MB run `options(shiny.maxRequestSize = 10 * 1024^2)`.

-   `type`: the "MIME type"[^action-transfer-1] of the file.
    This is a formal specification of the file type that is usually derived from the extension and is rarely needed in Shiny apps.

-   `datapath`: the path to where the data has been uploaded on the server.
    Treat this path as ephemeral: if the user uploads more files, this file may be deleted.
    The data is always saved to a temporary directory and given a temporary name.

[^action-transfer-1]: MIME type is short for "**m**ulti-purpose **i**nternet **m**ail **e**xtensions type".
    As you might guess from the name, it was originally designed for email systems, but now it's used widely across many internet tools.
    A MIME type looks like `type/subtype`.
    Some common examples are `text/csv`, `text/html`, `image/png`, `application/pdf`, `application/vnd.ms-excel` (excel file).

I think the easiest way to understand this data structure is to make a simple app.
Run the following code and upload a few files to get a sense of what data Shiny is providing.
You can see the results after I uploaded a couple of puppy photos (from Section \@ref(images)) in Figure \@ref(fig:upload).


```r
stopApp()
ui <- fluidPage(
  fileInput("upload", NULL, buttonLabel = "Upload...", multiple = TRUE),
  tableOutput("files")
)
server <- function(input, output, session) {
  output$files <- renderTable(input$upload)
}
```

<div class="figure">
<img src="demos/action-transfer/upload.png" alt="This simple app lets you see exactly what data Shiny provides to you for uploaded files. See live at &lt;https://hadley.shinyapps.io/ms-upload&gt;." width="672" />
<p class="caption">(\#fig:upload)This simple app lets you see exactly what data Shiny provides to you for uploaded files. See live at <https://hadley.shinyapps.io/ms-upload>.</p>
</div>

Note my use of the `label` and `buttonLabel` arguments to mildly customise the appearance, and use of `multiple = TRUE` to allow the user to upload multiple files.

### Uploading data {#uploading-data}

If the user is uploading a dataset, there are two details that you need to be aware of:

-   `input$upload` is initialised to `NULL` on page load, so you'll need `req(input$upload)` to make sure your code waits until the first file is uploaded.

-   The `accept` argument allows you to limit the possible inputs.
    The easiest way is to supply a character vector of file extensions, like `accept = ".csv"`.
    But the `accept` argument is only a suggestion to the browser, and is not always enforced, so it's good practice to also validate it (e.g. Section \@ref(validate)) yourself.
    The easiest way to get the file extension in R is `tools::file_ext()`, just be aware it removes the leading `.` from the extension.

Putting all these ideas together gives us the following app where you can upload a `.csv` or `.tsv` file and see the first `n` rows.
See it in action in <https://hadley.shinyapps.io/ms-upload-validate>.


```r
ui <- fluidPage(
  fileInput("upload", NULL, accept = c(".csv", ".tsv")),
  numericInput("n", "Rows", value = 5, min = 1, step = 1),
  tableOutput("head")
)

server <- function(input, output, session) {
  data <- reactive({
    req(input$upload)
    
    ext <- tools::file_ext(input$upload$name)
    switch(ext,
      csv = vroom::vroom(input$upload$datapath, delim = ","),
      tsv = vroom::vroom(input$upload$datapath, delim = "\t"),
      validate("Invalid file; Please upload a .csv or .tsv file")
    )
  })
  
  output$head <- renderTable({
    head(data(), input$n)
  })
}
```



Note that since `multiple = FALSE` (the default), `input$file` will be a single row data frame, and `input$file$name` and `input$file$datapath` will be a length-1 character vector.

## Download

Next, we'll look at file downloads, showing you the basic UI and server components, then demonstrating how you might use them to allow the user to download data or reports.

### Basics

Again, the UI is straightforward: use either `downloadButton(id)` or `downloadLink(id)` to give the user something to click to download a file.
The results are shown in Figure \@ref(fig:donwload-ui).


```r
ui <- fluidPage(
  downloadButton("download1"),
  downloadLink("download2")
)
```

<div class="figure">
<img src="demos/action-transfer/download.png" alt="A download button and a download link" width="600" />
<p class="caption">(\#fig:donwload-ui)A download button and a download link</p>
</div>

You can customise their appearance using the same `class` and `icon` arguments as for `actionButtons()`, as described in Section \@ref(action-buttons).

Unlike other outputs, `downloadButton()` is not paired with a render function.
Instead, you use `downloadHandler()`, which looks something like this:


```r
output$download <- downloadHandler(
  filename = function() {
    paste0(input$dataset, ".csv")
  },
  content = function(file) {
    write.csv(data(), file)
  }
)
```

`downloadHandler()` has two arguments, both functions:

-   `filename` should be a function with no arguments that returns a file name (as a string).
    The job of this function is to create the name that will be shown to the user in the download dialog box.

-   `content` should be a function with one argument, `file`, which is the path to save the file.
    The job of this function is to save the file in a place that Shiny knows about, so it can then send it to the user.

This is an unusual interface, but it allows Shiny to control where the file should be saved (so it can be placed in a secure location) while you still control the contents of that file.

Next we'll put these pieces together to show how to transfer data files or reports to the user.

### Downloading data

The following app shows off the basics of data download by allowing you to download any dataset in the datasets package as a tab separated file, Figure \@ref(fig:download-data).
I recommend using `.tsv` (tab separated value) instead of `.csv` (comma separated values) because many European countries use commas to separate the whole and fractional parts of a number (e.g. `1,23` vs `1.23`).
This means they can't use commas to separate fields and instead use semi-colons in so-called "c"sv files!
You can avoid this complexity by using tab separated files, which work the same way everywhere.


```r
ui <- fluidPage(
  selectInput("dataset", "Pick a dataset", ls("package:datasets")),
  tableOutput("preview"),
  downloadButton("download", "Download .tsv")
)

server <- function(input, output, session) {
  data <- reactive({
    out <- get(input$dataset, "package:datasets")
    if (!is.data.frame(out)) {
      validate(paste0("'", input$dataset, "' is not a data frame"))
    }
    out
  })
  
  output$preview <- renderTable({
    head(data())
  })
    
  output$download <- downloadHandler(
    filename = function() {
      paste0(input$dataset, ".tsv")
    },
    content = function(file) {
      vroom::vroom_write(data(), file)
    }
  )
}
```

<div class="figure">
<img src="demos/action-transfer/download-data.png" alt="A richer app that allows you to select a built-in dataset and preview it before downloading. See live at &lt;https://hadley.shinyapps.io/ms-download-data&gt;." width="600" />
<p class="caption">(\#fig:download-data)A richer app that allows you to select a built-in dataset and preview it before downloading. See live at <https://hadley.shinyapps.io/ms-download-data>.</p>
</div>

Note the use of `validate()` to only allow the user to download datasets that are data frames.
A better approach would be to pre-filter the list, but this lets you see another application of `validate()`.

### Downloading reports

As well as downloading data, you may want the users of your app to download a report that summarises the result of interactive exploration in the Shiny app.
This is quite a lot of work, because you also need to display the same information in a different format, but it is very useful for high-stakes apps.

One powerful way to generate such a report is with a [parameterised RMarkdown document](https://bookdown.org/yihui/rmarkdown/parameterized-reports.html).
A parameterised RMarkdown file has a `params` field in the YAML metadata:

``` {.yaml}
title: My Document
output: html_document
params:
  year: 2018
  region: Europe
  printcode: TRUE
  data: file.csv
```

Inside the document, you can refer to these values using `params$year`, `params$region` etc.
The values in the YAML metadata are defaults; you'll generally override them by providing the `params` argument in a call to `rmarkdown::render()`.
This makes it easy to generate many different reports from the same `.Rmd`.

Here's a simple example adapted from <https://shiny.rstudio.com/articles/generating-reports.html>, which describes this technique in more detail.
The key idea is to call `rmarkdown::render()` from the `content` argument of `downloadHander()`.
If you want to produce other output formats, just change the output format in the `.Rmd`, and make sure to update the extension (e.g. to `.pdf`).
See it in action at <https://hadley.shinyapps.io/ms-download-rmd>.


```r
ui <- fluidPage(
  sliderInput("n", "Number of points", 1, 100, 50),
  downloadButton("report", "Generate report")
)

server <- function(input, output, session) {
  output$report <- downloadHandler(
    filename = "report.html",
    content = function(file) {
      params <- list(n = input$n)
      
      id <- showNotification(
        "Rendering report...", 
        duration = NULL, 
        closeButton = FALSE
      )
      on.exit(removeNotification(id), add = TRUE)

      rmarkdown::render("report.Rmd", 
        output_file = file,
        params = params,
        envir = new.env(parent = globalenv())
      )
    }
  )
}
```























