+++
title = "DataTables from the DT package as a Shiny CRUD app interface"
date = 2019-03-10
updated = 2025-04-26
[taxonomies]
tags = ["R", "Shiny"]
+++

[Shiny Scorekeeper](https://github.com/hinkelman/Shiny-Scorekeeper) is a basketball scorekeeper app built with the [Shiny](https://shiny.posit.co) web framework for [R](https://www.r-project.org). I needed a new app for scoring video of my son's basketball games and I decided it would be a good learning experience to try to build my own. In this post, I describe using [DataTables from the DT package](https://rstudio.github.io/DT/) as the interface to the CRUD (create-read-update-delete) features in Shiny Scorekeeper.

<!-- more -->

As with many decisions when building Shiny Scorekeeper, I decided to bumble through creating my own CRUD components rather than follow [existing](https://github.com/bborgesr/wsds2017/tree/master/app) [examples](https://ipub.com/dev-corner/apps/shiny_crud01/). Eventually, I settled on a spreadsheet interface as a familiar, intuitive, and compact approach for creating teams and rosters in Shiny Scorekeeper. I had some previous experience using [rhandsontables](https://jrowen.github.io/rhandsontable/) as spreadsheets in Shiny apps but the DT package also recently added the option to [edit DataTables](https://blog.rstudio.com/2018/03/29/dt-0-4/) and I decided it would be fun to learn more about the capabilities of the DT package [[1]](#1).

Shiny Scorekeeper uses a homemade database comprised of a set of CSV files linked with ID columns. The teams table includes columns for TeamID, League, Team, and Season; the players table includes PlayerID, FirstName, and LastName; and, the rosters table includes TeamID, PlayerID, and Number (i.e., jersey number). The goal is to manage data for multiple players across many different teams and seasons.

### Teams

Teams.csv is read from file, stored as a reactive value (`rv[["teams"]]`), and rendered as a DataTable with custom options to simplify the display. Importantly, the TeamID column is disabled to make it inaccessible for editing. The proxy object (`proxyTeams`) allows for manipulation of the DataTable.

```
output$teamsTable <- renderDT({
  rv[["teams"]]
}, style = "default", rownames = FALSE, 
selection = "single", editable = list(target = "cell"),              
options = list(searching = FALSE, bPaginate = FALSE, info = FALSE,
                columnDefs = list(list(visible = FALSE, targets = c("TeamID")))))

proxyTeams <- dataTableProxy("teamsTable")
```

When the teams table is edited, the edited value replaces the previous value in `rv[["teams"]]` and the new `rv[["teams"]]` object replaces `proxyTeams`. 

```
observeEvent(input$teamsTable_cell_edit, {
  info <- input$teamsTable_cell_edit
  rv[["teams"]] <- edit_teams_row(rv[["teams"]], info$row, info$col + 1L, info$value)
  replaceData(proxyTeams, rv[["teams"]], resetPaging = FALSE, rownames = FALSE)
})

edit_teams_row = function(teams_table, row, col, value){
  if (col == 1) stop("TeamID column (col = 1) can't be updated")
  if (!is.character(value)) value = as.character(value)
  # check if value has any number of whitespace
  if (grepl("^\\s*$", value)) value = NA_character_
  teams_table[row, col] = value
  teams_table
}
```

A button to delete a row in the teams table is conditionally enabled when a row is selected in the teams table.

```
observe({
  req(nrow(rv[["teams"]]) > 0)
  state = if (is.null(input$teamsTable_rows_selected)) TRUE else FALSE
  updateActionButton(session, "delete_teams_row", disabled = state)
})
```

Deleting a row in the teams table also deletes the roster for that team and the players on that roster that are not on rosters for any other teams. When all the reactive values are updated, `proxyTeams` is also updated.

```
observeEvent(input$delete_teams_row,{
  req(input$teamsTable_rows_selected)
  tmp = delete_teams_row(rv[["teams"]], 
                          input$teamsTable_rows_selected,
                          rv[["players"]],
                          rv[["rosters"]])
  rv[["teams"]] <- tmp$teams_table
  rv[["players"]] <- tmp$players_table
  rv[["rosters"]] <- tmp$rosters_table
  replaceData(proxyTeams, rv[["teams"]], resetPaging = FALSE, rownames = FALSE) 
})

delete_teams_row = function(teams_table, teams_row, players_table = NULL, rosters_table = NULL){
  if (nrow(teams_table) == 0) stop("Can't delete row from empty teams_table")
  if (is.null(players_table)) players_table = init_players_table()
  if (is.null(rosters_table)) rosters_table = init_rosters_table()

  team_id = teams_table$TeamID[teams_row]
  new_teams_table = teams_table[-teams_row,]
  new_rosters_table = rosters_table[rosters_table$TeamID != team_id, ]
  new_players_table = players_table[players_table$PlayerID %in% new_rosters_table$PlayerID, ]

  list(teams_table = new_teams_table,
       players_table = new_players_table,
       rosters_table = new_rosters_table)
}
```

The add row button is not conditionally enabled. Adding a new row is always an option. The DT package includes a function (`addRow()`) for adding rows to a DataTable but it only works for client-side tables. 

```
observeEvent(input$add_teams_row,{
  rv[["teams"]] <- add_teams_row(rv[["teams"]])
  replaceData(proxyTeams, rv[["teams"]], resetPaging = FALSE, rownames = FALSE) 
})

add_teams_row = function(teams_table){
  rbind(teams_table,
        data.frame(TeamID = ids::random_id(),
                   League = NA_character_,
                   Team = NA_character_,
                   Season = NA_character_))
}
```

### Rosters

Creating the roster view is similar to the teams table. In this case, I am hiding both the `TeamID` and `PlayerID` columns. Also, Rosters.csv contains rosters for all teams and is stored in `rv[["rosters"]]` whereas `rv[["roster"]]` [[2]](#2) holds only the roster view for the team selected in the teams table.

```
output$rosterView <- renderDT({
  req(nrow(rv[["teams"]]) > 0, input$teamsTable_rows_selected, rv[["roster"]])
  rv[["roster"]]
}, style = "default", rownames = FALSE, 
selection = "single", editable = list(target = "cell"),                 
options = list(searching = FALSE, bPaginate = FALSE, info = FALSE,
                columnDefs = list(list(visible = FALSE, targets = c("TeamID", "PlayerID")))))

proxyRoster <- dataTableProxy("rosterView")

# roster is first assigned (and updated) when a row is selected in teamsTable
observeEvent(input$teamsTable_rows_selected,{
  req(nrow(rv[["teams"]]) > 0, input$teamsTable_rows_selected)
  team_id <- rv[["teams"]]$TeamID[input$teamsTable_rows_selected]
  rv[["roster"]] <- create_roster_view(team_id, rv[["players"]], rv[["rosters"]])
  replaceData(proxyRoster, rv[["roster"]], resetPaging = FALSE, rownames = FALSE)  
})

create_roster_view <- function(team_id, players_table, rosters_table){
  rosters_table |>
    filter(TeamID == team_id) |>
    left_join(players_table, by = join_by(PlayerID)) |>
    # maintain desired column order
    select(TeamID, PlayerID, FirstName, LastName, Number)
}
```

Editing a cell in the roster view involves jumping through a few extra hoops because two tables are being edited. As a reminder, the players table includes PlayerID, FirstName, and LastName and the rosters table includes TeamID, PlayerID, and Number. Because TeamID and PlayerID are hidden, only FirstName, LastName, and Number are editable. If first or last name are edited, then `rv[["players""]]` is updated. If jersey number is edited, then `rv[["rosters"]]` is updated. 

```
observeEvent(input$rosterView_cell_edit, {
  info <- input$rosterView_cell_edit
  tmp = edit_roster_row(rv[["roster"]], info$row, info$col + 1L, info$value,
                        rv[["players"]], rv[["rosters"]])
  rv[["players"]] <- tmp$players_table
  rv[["rosters"]] <- tmp$rosters_table
  rv[["roster"]] <- tmp$roster_view
  replaceData(proxyRoster, rv[["roster"]], resetPaging = FALSE, rownames = FALSE) 
})

edit_roster_row = function(roster_view, roster_row, roster_col, value, players_table, rosters_table){
  # don't use row and col as indices b/c updating underlying tables, not the view directly

  if (roster_col == 1) stop("TeamID column (roster_col = 1) can't be updated")
  if (roster_col == 2) stop("PlayerID column (roster_col = 2) can't be updated")

  if (!is.character(value)) value = as.character(value)
  # check if value has any number of whitespace
  if (grepl("^\\s*$", value)) value = NA_character_

  team_id = roster_view$TeamID[roster_row]
  player_id = roster_view$PlayerID[roster_row]
  col_name = colnames(roster_view)[roster_col]

  new_players_table = players_table
  if (col_name %in% colnames(players_table)){
    new_players_table[new_players_table$PlayerID == player_id, col_name] = value
  }

  new_rosters_table = rosters_table
  if (col_name %in% colnames(rosters_table)){
    new_rosters_table[new_rosters_table$TeamID == team_id &
                        new_rosters_table$PlayerID == player_id, col_name] = value
  }

  new_roster_view = create_roster_view(team_id, new_players_table, new_rosters_table)

  list(players_table = new_players_table,
       rosters_table = new_rosters_table,
       roster_view = new_roster_view)
}
```

The code for adding and deleting rows in the roster view is similar to the code for the teams table. You can find that code in the [GitHub repository](https://github.com/hinkelman/Shiny-Scorekeeper/blob/main/R/rosterServer.R).

When filling out a new roster, a dropdown menu allows for selecting from previously entered players. The dropdown is dynamically created with `renderUI()` because the contents of the dropdown depend on previous selections. 

```
output$previousPlayers <- renderUI({
  req(rv[["roster"]], rv[["players"]])
  
  rv_ids <- rv[["roster"]][["PlayerID"]]
  all_ids <- rv[["players"]][["PlayerID"]]
  # find PlayerIDs that haven't been added to roster
  ids <- all_ids[!(all_ids %in% rv_ids)] 
  
  req(!is.null(input$teamsTable_rows_selected) & length(ids) > 0) 

  d <- rv[["players"]] |> 
    filter(PlayerID %in% ids) |> 
    mutate(PlayerName = create_player_name(FirstName, LastName)) |> 
    filter(!is.na(PlayerName)) |> 
    arrange(FirstName)
  
  picker_ids <- setNames(d[["PlayerID"]], d[["PlayerName"]])
  
  layout_column_wrap(
    pickerInput("selected_players", "Select players",
                choices = picker_ids, multiple = TRUE, 
                options = pickerOptions(size = 7, `live-search` = TRUE)),
    actionButton("add_selected_players", "Add selected players", 
                  style = "margin-top: 32px;", icon = icon("plus-square"))
  )
})
```

Players selected in the dropdown are added to the roster with a button (`add_selected_players`). The players selected from the dropdown menu are appended to the bottom of `rv[["rosters"]]` and `rv[["roster"]]` is rebuilt. 

```
observeEvent(input$add_selected_players,{
  req(!is.null(input$teamsTable_rows_selected))
  
  team_id <- rv[["teams"]]$TeamID[input$teamsTable_rows_selected]
  
  rv[["rosters"]] <- bind_rows(rv[["rosters"]],
                                data.frame(TeamID = team_id,
                                          PlayerID = input$selected_players,
                                          Number = ""))
  
  rv[["roster"]] <- create_roster_view(team_id, rv[["players"]], rv[["rosters"]])
  replaceData(proxyRoster, rv[["roster"]], resetPaging = FALSE, rownames = FALSE)  
})
```

### Saving Data

A save button is enabled when changes are made to the tables and disabled when the save button is clicked. Clicking the save button updates the files on disk and the objects in the global environment. Comparing reactive values to objects in the global environment is used to conditionally enable/disable the save button.

```
observe({
  input$save_teams_roster_changes # take dependency on save button to disable button after saving
  save_state = (isTRUE(all.equal(teams, rv[["teams"]])) & 
                  isTRUE(all.equal(players, rv[["players"]])) & 
                  isTRUE(all.equal(rosters, rv[["rosters"]])))
  updateActionButton(session, "save_teams_roster_changes", disabled = save_state)
})

observeEvent(input$save_teams_roster_changes,{
  # write teams, rosters, & players from memory to disk
  write.csv(rv[["teams"]], file.path(data_dir, "Teams.csv"), row.names = FALSE)
  write.csv(rv[["players"]], file.path(data_dir, "Players.csv"), row.names = FALSE)
  write.csv(rv[["rosters"]], file.path(data_dir, "Rosters.csv"), row.names = FALSE)
  # update non-reactive versions to keep track of changes
  teams <<- rv[["teams"]]
  players <<- rv[["players"]]
  rosters <<- rv[["rosters"]]
})
```

### Conclusions

I'm satisfied with both the appearance and functionality of DataTables as a CRUD app interface for this hobby project. I'm less confident that my homemade database will hold up well with increasing amounts of data collected.

***

<a name="1"></a> [1] For a fancier interface to editable DataTables, check out the [DTEdit package.](https://github.com/jbryer/DTedit) 

<a name="2"></a> [2] I found the plural/singular distinction intuitive but it does make a small typo more likely to create a problem than if the names were longer.