<svelte:head>
</svelte:head>

<script>
    import { onMount } from 'svelte';
    import jQuery from 'jquery';
    import { API_KEY } from "./api_key.js";
    import {
        Dropdown,
        DropdownItem,
        DropdownMenu,
        DropdownToggle
    } from 'sveltestrap';

    // a bunch of variables that are needed across function scopes:
    let isDropdownOpen = false;
    let mounted = false;
    let basetime = 0;
    let date_picker = 0;
    let search_box = "";

    var cities_showing_tzs = [
      {"name": "Berlin", "tz": "Europe/Berlin", "lat": 52.517, "lng": 13.3889},
      {"name": "Sydney", "tz": "Australia/Sydney", "lat": -33.8549, "lng": 151.216},
    ];

    var cities_hiding_tzs = [
      {"name": "San Francisco", "tz": "America/Los_Angeles", "lat": -37.8142, "lng": 144.963},
      {"name": "Lisbon", "tz": "Europe/Lisbon", "lat": 38.7078, "lng": -9.13659},
      {"name": "Melbourne", "tz": "Australia/Sydney", "lat": -37.8142, "lng": 144.963},
    ];

    onMount(() => {
        // The page is ready.
        console.log('onMount');
        mounted = true;
        jQuery(document).ready(function(){
          console.log("document ready")
          allJsLoaded()});
        });

    async function newCitySelected(e) {
      console.log("newCitySelected")
      var city_name = e.suggestion.name
      var {lat, lng} = e.suggestion.latlng
      var new_city_data = await getNewCity(city_name, lat, lng);
      console.log(new_city_data)
      cities_showing_tzs = [...cities_showing_tzs, new_city_data];
      window.setTimeout(updateTzs, 200);
      localStorage.setItem("previousCitiesShown", JSON.stringify(cities_showing_tzs))
    }

    async function allJsLoaded() {
        console.log('allJsLoaded');
        date_picker = new Pikaday({ field: document.getElementById('datepicker'),
            format: 'D MMM YYYY',
            defaultDate: moment().toDate(),
            setDefaultDate: true,
            onSelect: function() {
                updateTzs(this.getMoment());}});

        var placesAutocomplete = places({
          appId: 'pl3XEH1COU1N',
          apiKey: '4bb455630afbb57b026c0b68a975de5f',
          container: document.querySelector('#address-input'),
          type: 'city'
        });
        placesAutocomplete.on('change', newCitySelected);

        jQuery(window).on('resize', windowResize);
        //If the user clicks on any item, hide the menu
        jQuery('#menuItems').on('click', '.dropdown-item', function(){
            jQuery("#dropdown_coins").dropdown('toggle');
        })
        setDefaultViewData();
        loadMomentElements();
        setTimeout(initialiseTable,100);   // for some reason, if we don't wait before calling this then the window size may get reported as `xs` instead of whatever size it actually is.
    };

    async function getNewCity(city_name, lat, lng) {
        var tzpromise = await gettz(city_name, lat, lng);
        return({"name": city_name, "tz": "", "lat": lat, "lng": lng, "clocks": []});
    }

    async function gettz(city_name, lat, lng) {
        const xhr = new XMLHttpRequest();
        xhr.open('GET', `https://maps.googleapis.com/maps/api/timezone/json?location=`+lat+`,`+lng+`&timestamp=1331766000&key=`+API_KEY);
        xhr.send();

        xhr.onload = () => {
          if (xhr.status >= 200 && xhr.status < 300) {
            var cityIndex = cities_showing_tzs.findIndex(a => ((a.name === city_name) && (a.lat === lat) && (a.lng === lng)));
            cities_showing_tzs[cityIndex].tz = JSON.parse(xhr.response).timeZoneId;
            return(JSON.parse(xhr.response).timeZoneId);
          } else {
            console.log("promise rejected")
            return("uhoh")
          }
        };
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

    let nb_clocks = 0;
    let datepicker_divs_count = 0;
    let datepicker_divs_width = 0;

    function getClockArray(clock_basetime) {
        let clock_array = [];
        clock_basetime.add(-Math.floor(nb_clocks/2)-1, "hours");
        for (var i = nb_clocks - 1; i >= 0; i--) {
            clock_array.push(clock_basetime.add(1, "hours").hours());
        }
        return clock_array;
    };

    function updateBasetime(clock, tz_row) {
        var new_basetime = basetime.add(cities_showing_tzs[0].clocks[clock]-basetime.hours(),"hours");
        console.log("----------------")
        console.log(clock)
        console.log(tz_row)
        console.log(cities_showing_tzs[tz_row].clocks[clock])
        console.log(cities_showing_tzs)
        console.log("----------------")
        updateTzs(new_basetime);
    };

    function clearHighlighting() {
      let class_name = ".timezone_clock";
      jQuery(class_name).css("font-weight", 'normal');
      jQuery(class_name).css("background", "none");
    }

    function defaultHighlighting() {
      clearHighlighting();
      let middle_col = (datepicker_divs_count).toString()
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
        let reference_tz = 0;

        for (var i = 0; i <= cities_showing_tzs.length - 1; i++) {
            if (i == 0) {
                // first element is the "reference timezone"
                reference_tz = basetime.tz(cities_showing_tzs[i].tz).utcOffset();
            }
            var local_time = moment(basetime.format()).tz(cities_showing_tzs[i].tz);
            var local_offset = local_time.utcOffset();
            var local_time_str = local_time.format();
            var offset_difference = (reference_tz - local_offset)/60
            if (offset_difference < 0) {
                var offset_difference_str = offset_difference.toString();
            } else {
                var offset_difference_str = "+" + offset_difference.toString();
            }
            cities_showing_tzs[i].clocks = getClockArray(local_time);
            localStorage.setItem("previousCitiesShown", JSON.stringify(cities_showing_tzs))
        }
    }

    function handleClick (ev, new_g) {
        ev.preventDefault();
        console.log("handleClick")
        var status = ev.target.attributes.status.value;
        if (status == "active") {
          console.log("active");
          console.log({"name": ev.target.attributes.city_name.value, "tz": ev.target.attributes.tz.value});
          var cityIndex = cities_showing_tzs.findIndex(a => ((a.name === ev.target.attributes.city_name.value) && (a.tz === ev.target.attributes.tz.value)));
          cities_hiding_tzs = [...cities_hiding_tzs, cities_showing_tzs[cityIndex]];
          cities_showing_tzs.splice(cityIndex,1)
          cities_showing_tzs = cities_showing_tzs;
        } else {
          console.log("inactive");
          console.log({"name": ev.target.attributes.city_name.value, "tz": ev.target.attributes.tz.value});
          var cityIndex = cities_hiding_tzs.findIndex(a => ((a.name === ev.target.attributes.city_name.value) && (a.tz === ev.target.attributes.tz.value)));
          cities_showing_tzs = [...cities_showing_tzs, cities_hiding_tzs[cityIndex]];
          cities_hiding_tzs.splice(cityIndex,1)
          cities_hiding_tzs = cities_hiding_tzs;
        }
        cities_hiding_tzs.sort((a, b) => {
          if (a.name > b.name) return 1;
          if (a.name < b.name) return -1;
        })
        cities_showing_tzs.sort((a, b) => {
          if (a.name > b.name) return 1;
          if (a.name < b.name) return -1;
        })
        console.log(cities_showing_tzs)
        console.log(cities_hiding_tzs)
        console.log("saving to local store")
        localStorage.setItem("previousCitiesShown", JSON.stringify(cities_showing_tzs))
        updateTzs(basetime);
        setTimeout(defaultHighlighting,50);
    }

    function setDefaultViewData() {
      let loaded_cities = "";
        if (localStorage.getItem("previousCitiesShown")) {
          loaded_cities = JSON.parse(localStorage.getItem("previousCitiesShown"));
          console.log("loading from local")
          cities_showing_tzs = loaded_cities
          for (var i = 0; i <= cities_showing_tzs.length - 1; i++) {
              var cityIndex = cities_hiding_tzs.findIndex(a => ((a.name === cities_showing_tzs[i].name) && (a.tz === cities_showing_tzs[i].tz)));
              if (cityIndex >= 0) {
                cities_hiding_tzs.splice(cityIndex,1)
                cities_hiding_tzs = cities_hiding_tzs;
              }
          }
        } else {
          console.log("no local storage found")
        }
    }

    function getViewportSize() {
        let viewportSize = jQuery("#sizer").find("div:visible").data("size");
        if ((viewportSize == "sm") || viewportSize == "xs") {
            nb_clocks = 3;
            datepicker_divs_count = 1;
            datepicker_divs_width = "20%";
        } else {
            nb_clocks = 9;
            datepicker_divs_count = 4;
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

    //// Dropdown menu search_box thing
    //Find every item inside the dropdown
    let items = document.getElementsByClassName("dropdown-item")

    function cityIsAlreadyOnPage(city_name) {
      return false;
    }


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
    .datepicker_table {
        background-color: inherit;
        border-width: 1px;
        /*border-style: solid;*/
        border-style: none;
        margin-bottom: 0;
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
    <table class="table datepicker_table" style:line-height:10vh>
        <tbody>
            <tr class="date_row">
                <td class="datepicker_arrows datepicker_arrows_left" on:click={() => datepickerArrowClicked("left")}>◄</td>
                <td class="datepicker_box"><input class="datepicker_input" type="text" id="datepicker"></td>
                <td class="datepicker_arrows datepicker_arrows_right" on:click={() => datepickerArrowClicked("right")}>►</td>
            </tr>
        </tbody>
    </table>
    <table class="table city_table" style:line-height:10vh>
        <tbody>
          {#if Array.isArray(cities_showing_tzs)}
              {#each cities_showing_tzs as cityobj,i}
                  <tr class="city_row">
                      <td class="city_name_active" city_name={cityobj.name} tz={cityobj.tz} status="active"  city_name_row={i} draggable={true}  on:click={event => handleClick(event, 0)}>
                        { cityobj.name }
                      </td>
                      {#if Array.isArray(cityobj.clocks)}
                        {#each cityobj.clocks as cl,j}
                            <td class="timezone_clock col_{j} clock_cell" style="width: {datepicker_divs_width}" on:mouseenter={event => hoverCol(j)} on:mouseleave={event => unHoverCol(j)} on:click={event => updateBasetime(j, i)}>
                                {cl}:00
                            </td>
                        {/each}
                      {/if}
                  </tr>
              {/each}

          {/if}
          {#if Array.isArray(cities_hiding_tzs)}
              {#each cities_hiding_tzs as cityobj,i}
                  <tr class="city_row">
                      <td class="city_name_inactive" city_name={cityobj.name} tz={cityobj.tz} status="inactive"  city_name_row={i} draggable={true}  on:click={event => handleClick(event, 0)}>
                          { cityobj.name }
                      </td>
                  </tr>
              {/each}
          {/if}
          <tr>
            <td colspan = "{nb_clocks +1}">
              <input type="search" id="address-input" placeholder="City search" />
            </td>
          </tr>
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
