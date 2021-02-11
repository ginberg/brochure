
<!-- README.md is generated from README.Rmd. Please edit that file -->

# brochure

<!-- badges: start -->

<!-- badges: end -->

**THIS IS A WORK IN PROGRESS, DO NOT USE**

The goal of `{brochure}` is to provide a mechanism for creating natively
multi-page `{shiny}` applications, *i.e* that can serve content on
multiple endpoints.

## Installation

You can install the released version of `{brochure}` with:

``` r
remotes::install_github("ColinFay/brochure")
```

## Example

Here is the minimal working example:

``` r
library(brochure)
library(shiny)

ui <- function(request){
  brochure(
    page(
      href = "/",
      ui = tagList(
        h1("This is my first page")
      )
    ),
    page(
      href = "/page2",
      ui =  tagList(
        h1("This is my second page")
      )
    )
  )
}

server <- function(
  input, 
  output, 
  session
){
  
}

brochureApp(ui, server)
```

Redirections can be used to redirect from one endpoint to the other:

``` r
ui <- function(request){
  brochure(
    page(
      href = "/",
      ui = tagList(
        h1("This is my first page")
      )
    ),
    redirect(
      from = "/nothere",
      to =  "/"
    )
  )
}

server <- function(
  input, 
  output, 
  session
){
  
}

brochureApp(ui, server)
```

A more elaborate example:

``` r
library(brochure)
library(shiny)

# Creating a navlink
nav_links <- tags$ul(
  tags$li(
    tags$a(href = "/", "home"), 
  ),
  tags$li(
    tags$a(href = "/page2", "page2"), 
  ),
  tags$li(
    tags$a(href = "/contact", "contact"), 
  )
)

ui <- function(request){
  brochure(
    # Pages
    page(
      href = "/",
      ui = tagList(
        h1("This is my first page"),
        nav_links,
        plotOutput("plota")
      )
    ),
    page(
      href = "/page2",
      ui =  tagList(
        h1("This is my second page"),
        nav_links,
        plotOutput("plotb")
      )
    ),
    page(
      href = "/contact",
      ui =  tagList(
        h1("Contact us"),
        nav_links,
        tags$ul(
          tags$li("Here"),
          tags$li("There")
        )
      )
    ),
    # Redirections
    redirect(
      from = "/page3",
      to =  "/page2"
    ),
    redirect(
      from = "/page4",
      to =  "/"
    )
  )
}

server <- function(
  input, 
  output, 
  session
){
  
  # THIS PART WILL BE RENDERED ON /
  output$plota <- renderPlot({
    print("In /")
    plot(mtcars)
  })
  
  # THIS PART WILL BE RENDERED ON /page2
  output$plotb <- renderPlot({
    print("In /page2")
    plot(airquality)
  })
  
}

brochureApp(ui, server)
```

## req\_handlers & res\_handlers

Each page, and the app, have a `req_handlers` and `res_handlers`
parameters, that can take a list of functions. A
req\_handlers/res\_handlers is a function that takes one or two
argument(s), `req` for req\_handlers and `req` & `res` for
res\_handlers. req\_handlers return `req` & res\_handlers return `res`,
potentially modified.

They can be used to register log, or to modify the objects, or any kind
of things you can think of. They are run when R is building the HTTP
response to send to the browser (i.e, no server code has been run yet),
following this process:

1.  R receives a `GET` request from the browser, creating a `req`
    object.
2.  The `req_handlers` are run using this `req`
3.  R creates an `httpResponse`, using this `req` and how you defined
    the UI
4.  The res\_handlers are run on this `httpResponse` (first app level
    res\_handlers, then page level res\_handlers)
5.  The `httpResponse` is returned to the browser

Note that if any req\_handlers returns an `httpResponse` object, it will
be returned to the browser immediately, without any further computation,
and are not passed to the `res_handlers`.

### Logging with `req_handlers()`

``` r
library(brochure)
library(shiny)

ui <- function(request){
  brochure(
    req_handlers = list(
      function(req){
        cli::cat_rule(
          sprintf("%s - %s", Sys.time(), req$PATH_INFO)
        )
        req
      }
    ),
    page(
      href = "/",
      ui = tagList(
        h1("This is my first page")
      ), 
      req_handlers = list(
        function(req){
          print("HOME")
          req
        }
      )
    ),
    page(
      href = "/page2",
      ui =  tagList(
        h1("This is my second page")
      )
    )
  )
}

server <- function(
  input, 
  output, 
  session
){
  
}

brochureApp(ui, server)
```

### Handling cookies using `res_handlers`

`res_handlers` can be used to set cookies, by adding a `Set-Cookie`
header.

Note that you can parse the cookie using `parse_cookie_string`.

``` r
parse_cookie_string( session$request$HTTP_COOKIE )
```

``` r
library(brochure)
library(shiny)

ui <- function(request){
  brochure(
    res_handlers = list(
      function(res, req){
        # If the query string is ?logout, we remove the cookie
        qs <- shiny::parseQueryString(req$QUERY_STRING)
        if ( length(qs) == 0 ){
          res$headers$`Set-Cookie` <- "plop=12; HttpOnly;"
        } else if ("logout" %in% names(qs)){
          res$headers$`Set-Cookie` <- "plop=12; Expires=Wed, 21 Oct 1950 07:28:00 GMT"
        }
        res
      }
    ),
    page(
      href = "/",
      ui = tagList(
        h1("This is my first page"), 
        tags$p("Try reloading this page with ?logout in the url to remove the Cookie"), 
        verbatimTextOutput("cookie1")
      )
    ),
    page(
      href = "/page2",
      ui =  tagList(
        h1("This is my second page"),
        verbatimTextOutput("cookie2")
      )
    ), 
    page(
      href = "/logout",
      ui = tagList(
        "Bye"
      ), 
      res_handlers = list( 
        function(res, req){
          res$headers$`Set-Cookie` <- "plop=12; Expires=Wed, 21 Oct 1950 07:28:00 GMT"
          res
        }
      )
    )
  )
}

server <- function(
  input, 
  output, 
  session
){
  
  output$cookie1 <- renderPrint({
    parse_cookie_string(
      session$request$HTTP_COOKIE
    )
  })
  
  output$cookie2 <- renderPrint({
     parse_cookie_string(
      session$request$HTTP_COOKIE
    )
  })
}

brochureApp(ui, server)
```

## Design pattern

Note that every time you open a new page, a **new shiny session is
launched**.

What that means is that there is no data persistence in R when
navigating from one page to the other. That might seem like a downside,
but I believe that it will actually be for the best: it will make
developers think more carefully about the data flow of their
application;

That being said, how do keep track of a user though pages, so that if
they do something in a page, it’s reflected on another?

To do that, you’d need to add a form of session identifier, like a
cookie: this can for example be done using the
[`{glouton}`](https://github.com/colinfay/glouton) package. You’ll also
need a form of backend storage (here in the example, we use
[`{cachem}`](https://github.com/r-lib/cachem), but you can also use an
external DB like SQLite or MongoDB).

``` r
library(brochure)
library(glouton)
library(shiny)
# Creating a storage system
cache_system <- cachem::cache_disk("inst/cache")

nav_links <- tags$ul(
  tags$li(
    tags$a(href = "/", "home"),
  ),
  tags$li(
    tags$a(href = "/page2", "page2"),
  ),
  tags$li(
    tags$a(href = "/contact", "contact"),
  )
)


ui <- function(request){
  brochure(
    # We add an extra dep to the brochure page, here {glouton}
    use_glouton(),
    page(
      href = "/",
      ui = tagList(
        h1("This is my first page"),
        nav_links,
        # The text enter on page 1 will be available on page 2, using
        # a session cookie and a storage system
        textInput("textenter", "Enter a text"),
        plotOutput("plota")
      )
    ),
    page(
      href = "/page2",
      ui =  tagList(
        h1("This is my second page"),
        nav_links,
        # The text enter on page 1 will be available here, reading
        # the storage system
        verbatimTextOutput("textdisplay"),
        plotOutput("plotb")
      )
    ),
    page(
      href = "/contact",
      ui =  tagList(
        h1("Contact us"),
        nav_links,
        tags$ul(
          tags$li("Here"),
          tags$li("There")
        )
      )
    )
  )
}

server <- function(
  input,
  output,
  session
){
  
  # THIS PART WILL BE RENDERED ON ALL PAGES
  
  # Enabling the brochure mechanism
  # brochure_enable()
  
  r <- reactiveValues()
  
  observeEvent(TRUE, {
    # Fetch the cookies using {glouton}
    r$cook <- fetch_cookies()
    
    # If there is no stored cookie for {brochure}, we generate it
    if (is.null(r$cook$brochure_cookie)){
      # Generate a random id
      session_id <- digest::sha1(paste(Sys.time(), sample(letters, 16)))
      # Add this id as a cookie
      add_cookie("brochure_cookie", session_id)
      # Store in in the reactiveValues list
      r$cook$brochure_cookie <- session_id
    }
    # For debugging purpose
    print(r$cook$brochure_cookie )
  },
  # We only need to do it once doing it once
  once = TRUE
  )
  
  # THIS PART WILL ONLY BE RENDERED ON /
  
  observeEvent( input$textenter , {
    # Use the session id to save on the cache system
    cache_system$set(
      paste0(
        r$cook$brochure_cookie,
        "text"
      ),
      input$textenter
    )
  })
  
  
  
  output$plota <- renderPlot({
    print("In /")
    plot(mtcars)
  })
  
  # THIS PART WILL ONLY BE RENDERED ON /page2
  
  output$textdisplay <- renderPrint({
    # Getting the content value based on the session cookie
    cache_system$get(
      paste0(
        r$cook$brochure_cookie,
        "text"
      )
    )
  })
  
  output$plotb <- renderPlot({
    print("In /page2")
    plot(airquality)
  })
  
}

brochureApp(ui, server)
```

## With golem

To adapt your `{golem}` based application to `{brochure}`, here are the
two steps to follow:

  - Build the ui with `brochure()` and `page()` in `app_ui()` :

<!-- end list -->

``` r
app_ui <- function(request) {
  tagList(
    # Leave this function for adding external resources
    golem_add_external_resources(),
    # Your application UI logic 
    brochure(
      page(
        href = "/",
        ui = mod_home_ui("home_ui_1") 
      ), 
      page(
        href = "/01",
        ui = mod_01_ui("01_ui_1")
      )
    )
  )
}
```

  - Replace `shinyApp` with `brochureApp` in `app_server()`:

<!-- end list -->

``` r
run_app <- function(
  onStart = NULL,
  options = list(), 
  enableBookmarking = NULL,
  ...
) {
  with_golem_options(
    app = brochureApp(
      ui = app_ui,
      server = app_server,
      onStart = onStart,
      options = options, 
      enableBookmarking = enableBookmarking
    ), 
    golem_opts = list(...)
  )
}
```
