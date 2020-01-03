<svelte:head>
</svelte:head>

<script>
    import { onMount } from 'svelte';
    import jQuery from "jquery";
    import { CITIES } from "./cities.js";

    import {
        Dropdown,
        DropdownItem,
        DropdownMenu,
        DropdownToggle
    } from 'sveltestrap';

    // a bunch of variables that are needed across function scopes:
    let isDropdownOpen = false;
    let mounted = false;
    let groups = [{},{},{},{}];
    let basetime = 0;
    let date_picker = 0;
    let search_box = "";

    groups[0] = {"name": "Visible_Cities_List", "city_names": []};
    groups[1] = {"name": "Cities_Showing_Timezones_List", "city_names": []};
    groups[2] = {"name": "All_Cities_List", "city_names": CITIES.city_names, "city_timezones": CITIES.city_timezones};
    groups[3] = {"name": "Cities_Shown_in_Dropdown_List", "city_names": []};

    onMount(() => {
        // The page is ready.
        console.log('onMount');
        mounted = true;
        allJsLoaded();
    });

    function allJsLoaded() {
        console.log('allJsLoaded');

        // Capture the event when user types into the search_box box
        search_box = document.getElementById("city_search_box_box");
        window.addEventListener('input', function () {
            filterCityDropdown(search_box.value.trim().toLowerCase());
        })

        date_picker = new Pikaday({ field: document.getElementById('datepicker'),
            format: 'D MMM YYYY',
            defaultDate: moment().toDate(),
            setDefaultDate: true,
            onSelect: function() {
                updateTzs(this.getMoment());}});

        setDefaultViewData();
        setSearchBoxData();
        loadMomentElements();
        setTimeout(initialiseTable,100);   // for some reason, if we don't wait before calling this then the window size may get reported as `xs` instead of whatever size it actually is.
    };

    function datepickerArrowClicked(direction) {
        let new_basetime = 0;
        let nbdays = -1;
        console.log("datepickerArrowClicked");
        if (direction == "right") {
            nbdays = 1;
        }
        new_basetime = basetime.add(nbdays, "days");
        console.log(new_basetime.format())
        date_picker.setMoment(new_basetime);
        updateTzs(new_basetime);
    }

    function initialiseTable() {
        console.log("initialiseTable");
        getViewportSize();
        updateTzs();
        setTimeout(defaultHighlighting,100);  // for some reason this doesn't happen unless we give it a timeout.
    }

    function loadMomentElements() {
        console.log('loadMomentElements');
        basetime = moment()
        console.log(basetime.format())
    };

    let tz_clocks = [0];
    let nb_clocks = 0;
    let datepicker_divs_count = 0;
    let datepicker_divs_width = 0;

    function getTimezoneName(city_name) {
        var index_in_all_city_list =  groups[2].city_names.indexOf(city_name);
        return groups[2].city_timezones[index_in_all_city_list];
    }

    function getClockArray(clock_basetime) {
        let clock_array = [];
        clock_basetime.add(-Math.floor(nb_clocks/2)-1, "hours");
        for (var i = nb_clocks - 1; i >= 0; i--) {
            clock_array = clock_array.concat(clock_basetime.add(1, "hours").hours());
        }
        return clock_array;
    };

    function updateBasetime(clock, tz_row) {
        var new_basetime = basetime.add(tz_clocks[0][clock]-basetime.hours(),"hours");
        updateTzs(new_basetime);
    };

    function clearHighlighting() {
      let class_name = ".timezone_clock";
      jQuery(class_name).css("font-weight", 'normal');
      jQuery(class_name).css("background", "none");
    }

    function defaultHighlighting() {
      clearHighlighting();
      let middle_col = (datepicker_divs_count+1).toString()
      let class_name = ".col_"+middle_col;
      jQuery(class_name).css("font-weight", 'bold');
      jQuery(class_name).css("background", "#eee");
    }


    function hoverCol(col_nb) {
      clearHighlighting();
      let class_name = ".col_"+col_nb;
      jQuery(class_name).css("font-weight", 'bold');
      jQuery(class_name).css("background", "#eee");
    };

    function unHoverCol(col_nb) {
      let class_name = ".col_"+col_nb;
      jQuery(class_name).css("font-weight", 'normal');
      jQuery(class_name).css("background", "none");
      defaultHighlighting();
    };

    function updateTzs(new_basetime) {
        console.log("updateTzs");

        if(typeof new_basetime === "undefined") {
            basetime = moment();
        } else {
            basetime = new_basetime
        }
        let tzs = [];
        tz_clocks = [];
        let reference_tz = 0;

        for (var i = 0; i <= groups[1].city_names.length - 1; i++) {
            if (i == 0) {
                // first element is the "reference timezone"
                reference_tz = basetime.tz(getTimezoneName(groups[1].city_names[i])).utcOffset();
            }
            var local_time = moment(basetime.format()).tz(getTimezoneName(groups[1].city_names[i]));
            var local_offset = local_time.utcOffset();
            var local_time_str = local_time.format();
            var offset_difference = (reference_tz - local_offset)/60
            if (offset_difference < 0) {
                var offset_difference_str = offset_difference.toString();
            } else {
                var offset_difference_str = "+" + offset_difference.toString();
            }
            tzs = tzs.concat(offset_difference_str + ": " + local_time_str);
            tz_clocks = tz_clocks.concat([getClockArray(local_time)]);
        }
    }

    function dragstart (ev, group, item) {
        ev.dataTransfer.setData("group", group);
        ev.dataTransfer.setData("item", item);
    }

    function dragover (ev) {
        ev.preventDefault();
        ev.dataTransfer.dropEffect = 'move';
    }

    function drop (ev, new_g) {
        ev.preventDefault();
        var i = ev.dataTransfer.getData("item");
        var old_g = ev.dataTransfer.getData("group");
        const city_name = groups[old_g].city_names.splice(i,1)[0];
        groups[new_g].city_names = groups[new_g].city_names.concat(city_name);
        groups[new_g].city_names.sort();
        updateTzs(basetime);
    }

    function handleClick (ev, new_g) {
        ev.preventDefault();
        console.log("handleClick")
        var i = ev.target.attributes.city_name_row.value;
        var old_g = ev.target.attributes.group.value;
        const city_name = groups[old_g].city_names.splice(i,1)[0];
        groups[new_g].city_names.push(city_name);
        groups[new_g].city_names.sort();
        search_box.value="";
        setSearchBoxData();
        updateTzs(basetime);
        setTimeout(defaultHighlighting,50);
    }

    function setDefaultViewData() {
        let cities_to_show = ['Buenos Aires', 'Riga', 'Sydney', 'Vancouver'];
        for (var i = 0; i <= cities_to_show.length - 1; i++) {
            groups[0].city_names = groups[0].city_names.concat(cities_to_show[i]);
        }
        groups[1].city_names = ["Berlin"];
    }

    function setSearchBoxData() {
        console.log("setSearchBoxData");
        var city_names = [];
        for (var i = 0; i <= groups[2].city_names.length - 1; i++) {
            if (!cityIsAlreadyOnPage(groups[2].city_names[i])) {
                // this is not already shown in the timezone list, so it can be added to the search_box list
                city_names = city_names.concat(groups[2].city_names[i]);
            }
        }
        groups[3].city_names = city_names;
    }

    function getViewportSize() {
        let viewportSize = jQuery("#sizer").find("div:visible").data("size");
        if ((viewportSize == "sm") || viewportSize == "xs") {
            nb_clocks = 5;
            datepicker_divs_count = 1;
            datepicker_divs_width = "16%";
        } else {
            nb_clocks = 9;
            datepicker_divs_count = 3;
            datepicker_divs_width = "10%";
        }
        return viewportSize;
    }

    function windowResize() {
        getViewportSize();
        console.log(getViewportSize());
        updateTzs();
        defaultHighlighting();
    }
    jQuery(window).on('resize', windowResize);


    //// Dropdown menu search_box thing
    //Find every item inside the dropdown
    let items = document.getElementsByClassName("dropdown-item")

    function cityIsAlreadyOnPage(city_name) {
        if ((groups[1].city_names.includes(city_name)) || (groups[0].city_names.includes(city_name))) {
            return true;
        }
        return false;
    }

    //For every word entered by the user, check if the symbol starts with that word
    //If it does show the symbol, else hide it
    function filterCityDropdown(word) {
        let collection = []
        let hidden = 0
        var city_names = [];

        for (let i = 0; i < groups[2].city_names.length; i++) {
            if (groups[2].city_names[i].toLowerCase().startsWith(word)) {
                if (!cityIsAlreadyOnPage(groups[2].city_names[i])) {
                    city_names = city_names.concat(groups[2].city_names[i]);
                }
            }
        }
        groups[3].city_names = city_names;
    }

    //If the user clicks on any item, hide the menu
    jQuery('#menuItems').on('click', '.dropdown-item', function(){
        jQuery("#dropdown_coins").dropdown('toggle');
    })


</script>

<style>
    .timezone_clock {
        align-items: center;
        border: none;
        text-align: center;
        vertical-align: middle;
    }
    .timezone_clock:hover {
        cursor: pointer;
    }
    .city_name_active {
        border-right-color: rgb(190,190,190);
        border-right-style: solid;
        border-right-width: 2px;
    }
    .city_name_inactive {
    border-right-color: rgb(190,190,190);
    border-right-style: solid;
    border-right-width: 2px;

      color: rgb(190,190,190);
      font-weight: lighter;
      background: inherit;
    }
    .inactive_row {
    }
    .city_name_inactive:hover {
        cursor: pointer;
    }
    .city_name_active:hover {
        cursor: pointer;
    }
    .city_table {
        background-color: inherit;
        border-width: 1px;
        /*border-style: solid;*/
        border-style: none;
        margin-bottom: 0;
        height:95vh;
    }
    .clock_cell {
        border-left-width: 1px;
        border-right-width: 1px;
        /*border-style: solid;*/
    }
    .city_search_row {
    }
    ::placeholder { /* Chrome, Firefox, Opera, Safari 10.1+ */
        opacity: 1; /* Firefox */
    }
    .search_box {
        background-color: inherit;
        border-width: 1px;
        box-shadow:none;
        padding: 0;
        margin: 0;
        border: none;
    }
    .highlight_clock {
        font-weight: bold;
        border-left-width: 2px;
        border-right-width: 3px;
    }
    .datepicker_box {
        vertical-align: middle;
        text-align: center;
        margin: 0px;
    }
    .datepicker_input {
        text-align: center;
        /*border: none;*/
        background-color: inherit;
    }
    .datepicker_input:hover {
        cursor: pointer;
    }
    .datepicker_arrows {
        vertical-align: middle;
    }
    .datepicker_arrows:hover {
        cursor: pointer;
    }
    .datepicker_arrows_left {
        text-align: right
    }
    .datepicker_arrows_right {
        text-align: left
    }
    .city_row {
        height:5vh;
    }
    .date_row {
        height:5vh;
    }
    .col {
      font-weight: lighter;
    }

</style>


<div class="container">
    <table class="table city_table" style:line-height:10vh>
        <tbody>
            <tr class="date_row">
                <td></td>
                <td class="datepicker_arrows datepicker_arrows_left" colspan={datepicker_divs_count} on:click={() => datepickerArrowClicked("left")}>◄</td>
                <td class="datepicker_box" colspan=3><input class="datepicker_input" type="text" id="datepicker"></td>
                <td class="datepicker_arrows datepicker_arrows_right" colspan={datepicker_divs_count} on:click={() => datepickerArrowClicked("right")}>►</td>
            </tr>
            {#each groups[1].city_names as city_name,i}
                <tr class="city_row" on:drop={event => drop(event, 1)} on:dragover={dragover}>
                    <td class="city_name_active" city_name_row={i} group=1 draggable={true} on:dragstart={event => dragstart(event, 1, i)}  on:click={event => handleClick(event, 0)}>
                        { city_name }
                    </td>
                    {#each tz_clocks[i] as cl,j}
                        <td class="timezone_clock col_{j} clock_cell" style="width: {datepicker_divs_width}" on:mouseenter={event => hoverCol(j)} on:mouseleave={event => unHoverCol(j)} on:click={event => updateBasetime(j, i)}>
                            {cl}:00
                        </td>
                    {/each}
                </tr>
            {/each}
            {#each groups[0].city_names as city_name,i}
            <tr class="city_row" on:drop={event => drop(event, 1)} on:dragover={dragover}>
            <td class="city_name_inactive" city_name_row={i} group=0 draggable={true} on:dragstart={event => dragstart(event, 0, i)} on:drop={event => drop(event, 0)} on:click={event => handleClick(event, 1)} on:dragover={dragover}>
                {city_name}
            </td>
            <td class="inactive_row" colspan={datepicker_divs_count}></td>
            <td class="inactive_row" colspan=3></td>
            <td class="inactive_row" colspan={datepicker_divs_count}></td>
            </tr>
            {/each}
<tr></tr>
            <tr class="city_row city_search_row">
                <td>
                    <Dropdown isOpen={isDropdownOpen} toggle={() => (isDropdownOpen = !isDropdownOpen)}>
                        <input type="search_box" class="form-control search_box" id="city_search_box_box" placeholder="Search City" on:click={() => (isDropdownOpen = !isDropdownOpen)}>
                        <DropdownMenu style="max-height: 40vh; overflow:scroll;">
                            {#each groups[3].city_names as menu_item,i}
                                <DropdownItem city_name_row={i} group=3 on:click={event => handleClick(event, 1)}>{menu_item}</DropdownItem>
                            {/each}
                        </DropdownMenu>
                    </Dropdown>
                </td>
                {#each tz_clocks[0] as clock,i}
                    <td></td>
                {/each}
            </tr>
            <tr></tr>
        </tbody>
    </table>
</div>

<div id="sizer">
    <div class="d-block d-sm-none d-md-none d-lg-none d-xl-none" data-size="xs"></div>
    <div class="d-none d-sm-block d-md-none d-lg-none d-xl-none" data-size="sm"></div>
    <div class="d-none d-sm-none d-md-block d-lg-none d-xl-none" data-size="md"></div>
    <div class="d-none d-sm-none d-md-none d-lg-block d-xl-none" data-size="lg"></div>
    <div class="d-none d-sm-none d-md-none d-lg-none d-xl-block" data-size="xl"></div>
</div>
