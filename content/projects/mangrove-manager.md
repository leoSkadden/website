+++
title = "Mangrove Manager"
date = 2024-04-20
[extra]
toc = true
+++

**🚧Project in progress🚧**

The Mangrove Manager project is a 100% Rust tool to manage a mangrove nursery operation across multiple locations for a local non-profit. Beyond the basic functionality of the site the primary design consideration is to create a system that is stable and easy to maintain. A secondary object of the project is to explore how feasible it is to build a production grade frontend using Rust compiled to WASM.

## Background

The Mangrove Manager started life as an upgrade to the nursery that my wife runs for [Marine Resources Council (MRC)](https://lovetheirl.org), a local non-profit devoted to the conservation of the [Indian River Lagoon](https://en.wikipedia.org/wiki/Indian_River_Lagoon). They run a state-licensed mangrove nursery program to grow the trees from propagules (similar to seeds, propagules are dropped by mature trees to reproduce) to ultimately be used in coastline restoration projects around Florida. The nursery is split between two locations, a smaller nursery at the MRC main office and a larger nursery at a conservation property managed by MRC. By implementing this system we are hoping to introduce more automation in the management of the nursery and provide efficiency insights through data.

## System Requirements

### Basic Functionality

The basic functionality of the system is to track individual plants by assigning an ID number that can be encoded on a barcode or QR code and attached to the plant. In order to add new plants they will scan the ID codes and create new entries in the database. Plants will be able to be viewed and modified either by navigating the web interface or by scanning an ID code on the plant.

Also of note in the basic requirements is that this system is about as un-scaled as you could imagine. All users of the system will be in the same state, the total number of concurrent users possible is around 10 and the most likely maximum number of concurrent users is 3. 97% of the time the system will only have a single user.

### Data Model

The data points that will be collected for each plant are as follows:

- ID
  - The value that is encoded on the plant's barcode
- Nursery
  - Which physical location is this plant currently at?
  - Sometimes plants need to be moved between nurseries or inventory levels need to be checked for scheduling pick ups.
- Pot Size
  - How many gallons is the pot that this plant is in?
  - Plants are sold based on pot size, as the tree grows it get's moved into larger pots.
- Date Added
  - When was this plant added to the nursery?
  - This date can be used to give a rough, but plenty precise enough, estimate of plant age. New plants are brought into the nursery as propagules and are eventually put into a one gallon pot, at which point they are given an ID tag and entry in the system.
- Buyer
  - Does this plant already have a buyer?
- Present in Nursery
  - Occasionally plants are pulled out of the nurseries either for pickup or to be brought to an event as temporary decorations. In either case it's helpful to know that the plant is not in the nursery proper.
- Notes
  - Any notes left by nursery staff

## Tech Stack

As stated before this project is 100% Rust (ok maybe more like 95% if we include HTML and CSS). I was fortunate enough to find a project template created by Robert Krahn that was using the two frameworks that had made there way to my short list. You can read about the template on his blog [here](https://robert.kra.hn/posts/2022-04-03_rust-web-wasm/) ([archived](https://web.archive.org/web/20240425200159/https://robert.kra.hn/posts/2022-04-03_rust-web-wasm/)) or checkout the github page [here](https://github.com/rksm/axum-yew-setup).

{{ mangrove_manager_system_diagram() }}

### Shortlist

For the API layer I'm using [Axum](https://github.com/tokio-rs/axum), a web framework from the Tokio project utilizing web primitives from Tower and Hyper. Axum seemed like the best maintained and most ergonomic choice out of the available libraries.

For the frontend I wound up choosing [Yew](https://yew.rs). While there are plenty of Rust frontend frameworks to choose, [Dioxus](https://dioxuslabs.com) was a close second choice, Yew won out due to it's position as an established framework with many contributors. Yew also has an order of magnitude more downloads than Dioxus at the time of writing. 

The final major component of the tech stack was to pick out a library or ORM for the data layer. I prefer to make that kind of choice after I have already settled on a database system, to ensure an acceptable Developer Experience. Normally my go to solution for databases is Postgres, however in this case I felt that it would be the wrong choice. This system is intended to have minimal maintenance, adding a full database system means additional work to keep it up to date. It also complicates the hosting requirements of the system, either the host will need enough resources to run the database and web server or the database will need its own host. Instead I decide to go with SQLite. I have no concerns around hitting a performance bottle neck in this system, the only addition to the deployment will be to ensure that the SQLite package is installed on the host, and it makes backups trivial.

With a database system chosen I was able to review the libraries available. In the end I decided to go with [SQLx](https://github.com/launchbadge/sqlx), primarily because I was interested in exploring the features of their sql macro. This macro accepts normal SQL statements as well as any data that needs to be bound to the statement and provides intellisense (via LSP builds) and build time validation of your SQL statements. This validation goes beyond a simple syntax check and can actually check your queries against your own database scheme! That means it can tell you that your query is invalid because it is trying to select from the column `pot_szei` which doesn't exist in your database 🤯.

## Minimal Viable Product

As there is currently no management system in place for the nurseries any system that can meet the basic functionality requirements laid out above will be a vast improvement. In order to deliver value as soon as possible I'm attempting to be dogmatic about creating a minimally viable product. I've identified a few key areas of work that will establish patterns or underlying infrastructure that will be needed before I can start focusing on the basic functionality.

### Pre-MVP Validation

- Data layer
  - Establishing the design of the database access layer will affect how all the rest of the code is structured. For systems like this with limited expansion of the data model and no surprise changes coming from other teams I favor a Data Access Object pattern over something like an ORM. This means creating an object for each table with methods that abstract the SQL statements being run.
- Logical data models
  - It's very difficult to write Rust code without establishing your basic types ahead of time. The logical data models are the basic unit of the API and Frontend. These will evolve over time, the focus is on creating a type that can compile rather than a type that is completely correct.
- Basic functionality of Yew
  - I want to verify a few basic aspects of Yew are working for this architecture. If Yew is going to be difficult to work with I'll swap it out in favor of a JS framework to keep the project moving.
  - Axum to Yew routing
    - The frontend URLs are first handled by Axum and sent to the Yew site if the URL doesn't match an established API endpoint. I want to verify that I haven't broken the template and this routing is still working before moving onto more complex tests.
  - Building a basic home page
    - This is a test to familiarize myself with the basics of Yew and get some simple HTML onto the page. This test is focused on the ergonomics and documentation that Yew provides.
  - View Plants
    - The final test of Yew was to build the simplest possible page to start working with data. I split this test into a few parts of increasing complexity. The webpage is a simple data table to display the Plants that have been added to the system.
      1. Create a Yew component that outputs a full row of the table using a `Plant` model that is constructed in the frontend (not including any API calls yet)
      2. Refactor the component to output an arbitrary number of rows
      3. Make an API call for a single plant and display that as a row of the table
      4. Make an API call for a full page of the table and display all rows

### MVP Goals

The rest of the MVP will consist of the following features:

- Print ID codes
  - When a group of plants are ready to be added to the nursery this site needs to provide a way to print off a set of valid IDs.
- Bulk add plants
  - When adding groups of plants, every data point other than `ID` will remain static for the entire session (newly potted plants are all going in the same size pots with the same nursery location). This mode also needs to be designed in a way that is conducive to the physical workflow being used, namely scanning ID tags that are hung on the plants.
- Create, Read, and Update capabilities for plants and nurseries
  - For the MVP any deleting of records can be handled manually through the database's interface (post-MVP a soft delete and table archiving will be added). Basic create, read, and update functionality in the API and the web UI will be needed from day one.
