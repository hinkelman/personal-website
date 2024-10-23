+++
title = "DataTables from the DT package as a Shiny CRUD app interface"
date = 2019-03-10
updated = 2023-03-27
[taxonomies]
tags = ["R", "Shiny"]
+++

[Shiny Scorekeeper](https://github.com/hinkelman/Shiny-Scorekeeper) is a basketball scorekeeper app built with the [Shiny](https://shiny.rstudio.com) web framework for [R](https://www.r-project.org). I needed a new app for scoring video of my son's basketball games and I decided it would be a good learning experience to try to build my own. In this post, I describe using [DataTables from the DT package](https://rstudio.github.io/DT/) as the interface to the CRUD (create-read-update-delete) features in Shiny Scorekeeper. The post assumes familiarity with features of Shiny apps, particularly [`reactiveValues()`](https://shiny.rstudio.com/reference/shiny/0.14/reactiveValues.html), [`observe()`](https://shiny.rstudio.com/reference/shiny/latest/observe.html), and [`observeEvent()`](https://shiny.rstudio.com/reference/shiny/1.0.1/observeEvent.html).

<!-- more -->

As with many decisions when building Shiny Scorekeeper, I decided to bumble through creating my own CRUD components rather than follow [existing](https://github.com/bborgesr/wsds2017/tree/master/app) [examples](https://ipub.com/dev-corner/apps/shiny_crud01/). Eventually, I settled on a spreadsheet interface as a familiar, intuitive, and compact approach for creating teams and rosters in Shiny Scorekeeper. I had some previous experience using [rhandsontables](https://jrowen.github.io/rhandsontable/) as spreadsheets in Shiny apps but the DT package also recently added the option to [edit DataTables](https://blog.rstudio.com/2018/03/29/dt-0-4/) and I decided it would be fun to learn more about the capabilities of the DT package [[1]](#1).

Shiny Scorekeeper uses a homemade database comprised of a set of CSV files linked with ID columns. The teams table includes columns for TeamID, Season, League, and Team; the players table includes PlayerID, FirstName, and LastName; and, the rosters table includes TeamID, PlayerID, and Number (i.e., jersey number). The goal is to manage data for multiple players across many different teams and seasons.

### Teams

Teams.csv is read from file, stored as a reactive value (`rv[["teams"]]`), and rendered as a DataTable with custom options to simplify the display. Importantly, the TeamID column is disabled to make it inaccessible for editing. The proxy object (`proxyTeams`) allows for manipulation of the DataTable.

```
  output$teamsTable <- renderDT(
    rv[["teams"]], selection = "single", style = "bootstrap", rownames = FALSE,
    editable = list(target = "cell", disable = list(columns = 0)),              # disable TeamID
    options = list(searching = FALSE, bPaginate = FALSE, info = FALSE))
  
  proxyTeams <- dataTableProxy("teamsTable")

```
When the `teamsTable` is edited, the edited value replaces the previous value in `rv[["teams"]]` and the new `rv[["teams"]]` object replaces `proxyTeams`. The `coerceValue()` function from the DT package coerces the edited value (passed as character string) to the type of the value that it is replacing.

```
observeEvent(input$teamsTable_cell_edit, {
  info = input$teamsTable_cell_edit
  i = info$row
  j = info$col + 1L  # column index offset by 1
  v = info$value
  rv[["teams"]][i, j] = coerceValue(v, rv[["teams"]][i, j])
  replaceData(proxyTeams, rv[["teams"]], resetPaging = FALSE, rownames = FALSE) 
})

```
A button to delete a row in the `teamsTable` is conditionally shown and hidden with [`toggle()`](https://rdrr.io/cran/shinyjs/man/visibilityFuncs.html) from the [shinyjs package](https://deanattali.com/shinyjs/).

```
observe({
  toggle("delete_teams_row", 
         condition = nrow(rv[["teams"]]) > 0 & !is.null(input$teamsTable_rows_selected))
})

```
Deleting a row in the `teamsTable` also deletes the roster for that team and the players on that roster that are not on rosters for any other teams. When all the reactive values are updated, `proxyTeams` is also updated.

```
observeEvent(input$delete_teams_row,{
  req(input$teamsTable_rows_selected)
  i = input$teamsTable_rows_selected
  
  rv[["rosters"]] = rv[["rosters"]] %>% 
    filter(TeamID != rv[["teams"]]$TeamID[i])  # drop old roster
  rv[["players"]] = rv[["players"]] %>% 
    filter(PlayerID %in% rv[["rosters"]][["PlayerID"]]) # drop players not on any rosters
  rv[["teams"]] <- rv[["teams"]][-i,]  # needs to come last b/c updating rv$rosters requires rv$teams
  
  replaceData(proxyTeams, rv[["teams"]], resetPaging = FALSE, rownames = FALSE)  
})

```
The add row button is not shown conditionally. Adding a new row is always an option. The DT package includes a function (`addRow()`) for adding rows to a DataTable but it only works for client-side tables. 

In my homemade database, I include single-column CSV files for tracking unique IDs for teams and players (IDs are integers). Initially, I created an ID by finding the max ID in either the teams or players tables, but I started to worry that I could get unexpected behavior with lots of row addition and deletion. Rather than thinking through whether that was a legitimate concern, I added the clunky solution of single-column CSV files for ID tracking. Those "tables" (`teamIDs` and `playerIDs`) are not stored in reactive values; the tables are updated in memory with the super assignment operator (`<<-`) and updated on disk with `write.csv()`. A new row is added as a list with empty strings `""` or `NA_integer_` as placeholder values. 

```
observeEvent(input$add_teams_row,{
  #addRow() only works when server = FALSE
  req(rv[["teams"]])
  
  # update master list of team IDs
  tid <- nrow(teamIDs) + 1L # ID and row number are the same
  teamIDs[tid,] <<- tid
  write.csv(teamIDs, file.path(data_fldr, "TeamIDs.csv"), row.names = FALSE)
  
  # update master list of player IDs
  pid <- nrow(playerIDs) + 1L # ID and row number are the same
  playerIDs[pid,] <<- pid
  write.csv(playerIDs, file.path(data_fldr, "PlayerIDs.csv"), row.names = FALSE)
  
  # update all of the relevant tables
  ti <- nrow(rv[["teams"]]) + 1L
  rv[["teams"]][ti,] <- list(tid, "", "", "")
  ri <- nrow(rv[["rosters"]]) + 1L
  rv[["rosters"]][ri,] <- list(tid, pid, NA_integer_)
  pi <- nrow(rv[["players"]]) + 1L
  rv[["players"]][pi,] <- list(pid, "", "")
  replaceData(proxyTeams, rv[["teams"]], resetPaging = FALSE, rownames = FALSE)  # important
})
```

### Rosters

Creating the `rosterTable` is similar to the `teamsTable`. In this case, I am hiding the `TeamID` column (indexed at 0) and disabling the PlayerID column (indexed at 1). Also, Rosters.csv contains rosters for all teams and is stored in `rv[["rosters"]]` whereas `rv[["roster"]]` [[2]](#2) holds only the roster for the team selected in the `teamsTable`.

```
output$rosterTable <- renderDT(
  rv[["roster"]], selection = "single", style = "bootstrap", rownames = FALSE,
  editable = list(target = "cell", disable = list(columns = 1)),              # disable PlayerID column
  options = list(searching = FALSE, bPaginate = FALSE, info = FALSE,
                  columnDefs = list(list(visible = FALSE, targets = 0))))      # hide TeamID column

proxyRoster <- dataTableProxy("rosterTable")

observeEvent(input$teamsTable_rows_selected,{
  req(input$teamsTable_rows_selected)
  ti <- rv[["teams"]]$TeamID[input$teamsTable_rows_selected]
  rv[["roster"]] <- filter(rv[["rosters"]], TeamID == ti) %>% 
    left_join(rv[["players"]], by = "PlayerID")
  replaceData(proxyRoster, rv[["roster"]], resetPaging = FALSE, rownames = FALSE)  # important
})

```
Editing a cell in the `rosterTable` involves jumping through a few extra hoops because two tables are being edited. As a quick reminder, the players table includes PlayerID, FirstName, and LastName and the rosters table includes TeamID, PlayerID, and Number. Because TeamID is hidden and PlayerID is disabled, only FirstName, LastName, and Number are editable. If first or last name are edited, then `rv[["players""]]` is updated. If jersey number is edited, then `rv[["rosters"]]` is updated. 

```
observeEvent(input$rosterTable_cell_edit, {
  info = input$rosterTable_cell_edit
  i = info$row
  j = info$col + 1L  # column index offset by 1
  v = info$value
  
  # get IDs for row where change was made
  tid = rv[["roster"]][["TeamID"]][i]
  pid = rv[["roster"]][["PlayerID"]][i]
  
  # find row indices
  ri = which(rv[["rosters"]][["TeamID"]] == tid & rv[["rosters"]][["PlayerID"]] == pid)
  pi = which(rv[["players"]][["PlayerID"]] == pid)
  
  # find colunm name
  cn = names(rv[["roster"]])[j]
  
  # update values
  if (cn == "Number"){ 
    rv[["rosters"]][ri, cn] = coerceValue(v, rv[["rosters"]][ri, cn])
  }else{
    rv[["players"]][pi, cn] = coerceValue(v, rv[["players"]][pi, cn])
  }
  
  rv[["roster"]] = rv[["rosters"]] %>% # rebuild rv$roster
    filter(TeamID == tid) %>% 
    left_join(rv[["players"]], by = "PlayerID")
  
  replaceData(proxyRoster, rv[["roster"]], resetPaging = FALSE, rownames = FALSE)  # important
})
```

The code for adding and deleting rows in the `rosterTable` is very similar to the code for the `teamsTable`. Interested readers can find that code in the [server.R file in the GitHub repository](https://github.com/hinkelman/Shiny-Scorekeeper/blob/master/server.R).

When filling out a new roster, a dropdown menu allows for selecting from previously entered players. The dropdown is dynamically created with `renderUI()` because the contents of the dropdown depend on previous selections.

```
first_last <- function(first, last){
  ifelse(last == "", first, paste(first, last))
}

output$previousPlayers <- renderUI({
  req(rv[["roster"]], rv[["players"]])
  
  sel_ids <- rv[["roster"]][["PlayerID"]]
  all_ids <- rv[["players"]][["PlayerID"]]
  ids <- all_ids[!(all_ids %in% sel_ids)] # find PlayerIDs that haven't been added to roster
  
  req(length(ids) > 0) # at least one player that could be selected
  
  d <- rv[["players"]] %>% 
    filter(PlayerID %in% ids) %>%
    mutate(PlayerName = first_last(FirstName, LastName)) %>% 
    arrange(FirstName)
  
  picker.ids <- d[["PlayerID"]]
  names(picker.ids) <- d[["PlayerName"]]
  pickerInput("selected_players", "Select previous players", 
              choices = picker.ids, multiple = TRUE,
              options = list(`live-search` = TRUE))
})
```

Players selected in the dropdown are added to the roster with a button (`add_selected_players`) that is conditionally shown or hidden based on the existence of the dropdown menu.  

```
observe({
  toggle("add_selected_players", condition = !is.null(input$selected_players))
})
```

The players selected from the dropdown menu are appended to the bottom of `rv[["rosters"]]` and `rv[["roster"]]` is rebuilt. 

```
observeEvent(input$add_selected_players,{
  req(rv[["roster"]], rv[["players"]]) # probably not necessary b/c handled upstream
  
  tid <- rv[["roster"]]$TeamID[1] # all rows in rv[["roster"]] have same TeamID
  
  rv[["rosters"]] <- bind_rows(rv[["rosters"]],
                                data.frame(TeamID = tid, 
                                          PlayerID = as.integer(input$selected_players), 
                                          Number = NA_integer_,
                                          stringsAsFactors = FALSE))
  
  rv[["roster"]] <- rv[["rosters"]] %>% # rebuild rv$roster
    filter(TeamID == tid) %>% 
    left_join(rv[["players"]], by = "PlayerID")
  replaceData(proxyRoster, rv[["roster"]], resetPaging = FALSE, rownames = FALSE)
})
```
### Saving Data

A save button is shown when changes are made to the tables and hidden when the save button is clicked. Clicking the save button updates the files on disk and the objects in the global environment. Comparing reactive values to objects in the global environment is used to conditionally show/hide the save button.

```
observe({
  input$save_teams_roster_changes # take dependency on save button to hide button after saving
  toggle("save_teams_roster_changes", condition = !isTRUE(all_equal(players, rv[["players"]])) | !isTRUE(all_equal(rosters, rv[["rosters"]])) | !isTRUE(all_equal(teams, rv[["teams"]])))
})

observeEvent(input$save_teams_roster_changes,{
  # write teams, rosters, & players from memory to disk
  write.csv(rv[["teams"]], paste0(data_fldr, "Teams.csv"), row.names = FALSE)
  write.csv(rv[["rosters"]], paste0(data_fldr, "Rosters.csv"), row.names = FALSE)
  write.csv(rv[["players"]], paste0(data_fldr, "Players.csv"), row.names = FALSE)
  # update non-reactive versions to keep track of changes
  teams <<- rv[["teams"]]
  rosters <<- rv[["rosters"]]
  players <<- rv[["players"]]
})
```
### Conclusions

I'm satisfied with both the appearance and functionality of DataTables as a CRUD app interface for this hobby project. In fact, in a different part of the app, I even use DataTables in place of a `selectInput` because I wanted the option to sort by different fields when selecting records for display. I'm less confident that my homemade database will hold up well with increasing amounts of data collected.

***

<a name="1"></a> [1] For a fancier interface to editable DataTables, check out the [DTEdit package.](https://www.bryer.org/post/2018-22-26-dtedit/) 

<a name="2"></a> [2] I should have chosen better names. I found the plural/singular distinction intuitive but it does make a small typo more likely to create a problem than if the names were longer.