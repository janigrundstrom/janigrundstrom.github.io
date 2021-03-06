
<!DOCTYPE HTML>
<html>
<head>
<meta http-equiv="content-type" content="text/html" charset="UTF-8">
<meta name="viewport" content="width=device-width"/>

<!--
    Mass and balance calculator for "OH-COK"

    https://github.com/kaltsi/Mass-and-Balance

    Copyright (c) 2012 - 2014 Juha Kallioinen <kaltsi+github@gmail.com>

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in
    all copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
    THE SOFTWARE.

-->

<style type="text/css">
table.a {
    border-radius: 4%;
    padding: 10px 10px 10px 10px;
    box-shadow: 3px 3px 5px gray;
    border-spacing:4px;
}

table.a th {
    border: 0px;
    padding: 5px 5px 5px 0px;
    text-align: right;
}

table.a td {
    text-align: right;
    padding: 3px;
    box-shadow: 2px 2px 3px gray;
}

.alignleft {
    float: left;
}

.alignright {
    float: right;
}

.aligncenter {
    text-align: center;
}

.number_class {
    position: absolute;
    border: 1px solid;
    border-radius: 15px;
    box-shadow: 3px 3px 2px rgba(0,0,0,0.4);
    background-color: #83ffff;
    padding: 4px;
    width: 3em;
    text-align: right;
    z-index: 10;
    padding-right: 10px;
}

input[type=number]::-webkit-inner-spin-button, 
input[type=number]::-webkit-outer-spin-button {
    -webkit-appearance: none;
    margin: 0;
}

input[type=number] {
    -moz-appearance: textfield;
    border: 0px;
    background-color: #83ffff;
    width: 2em;
    text-align: right;
    font-size: 1.2em;
    font-family: serif;
}
</style>

<script type="text/javascript">

    /*  Some conversion constants */

    var g_kg_lbs        = 2.20462262; // 1 kg  -> lb
    var g_ltr_usg       = 0.26417205; // 1 ltr -> USG
    var g_m_inch        = 39.3700787; // 1 m   -> inches
    var g_mass_unit     = "kg";       // default in kilos

    /* defaults and storage object */

    var g_defs = {
	acft      : "OH-COK",
	acft_type : "Cessna C172N",
	origin    : "Matkatavaratilojen suurin yhteenlaskettu kokonaismassa 54 kg",
	ltr2kg    : 0.72,
	usg2lb    : 0, // will be computed at setup()
	s : { // These are the default slider values for this plane
	    FRONT_SEAT : 150,
BACK_SEAT : 0,
FUEL : 189,
BAGGAGE : 8,
BAGGAGE2 : 0,
TAXI_FUEL : 5,
FUEL_FLOW : 32,
FLIGHT_TIME : 60,

	    mass  : 0,     // 0 = kg, 1 = lb
	    lang  : 0      // 0 = English, 1 = Suomi
	},
	debug : false,
    };

    /* mass and balance envelope limits (metres and kilos) */
    
    var g_lim = {
	x          : [0.889, 0.889, 0.978, 1.201, 1.201],
	y          : [644, 884, 1043, 1043, 644],
	y_land     : [644, 884, 1043, 1043, 644],
	x_draw_max : 475,
	y_draw_max : 375,
	x_draw_min : 1,
	y_draw_min : 1,
	lim_lnd1   : /*LND_LIMIT_X1*/  0.8075  /**/,
	lim_lnd2   : /*LND_LIMIT_X2*/  0.925   /**/
    };

    g_lim.min_arm  = Math.min.apply(Math, g_lim.x);
    g_lim.max_arm  = Math.max.apply(Math, g_lim.x);
    g_lim.min_mass = Math.min.apply(Math, g_lim.y);
    g_lim.max_mass = Math.max.apply(Math, g_lim.y);

    /* change the unit values for the limits */

    g_lim.unitchg = function lim_unit_change()
    {
	function convert(a, m) {
	    var i;
	    for (i = 0; i < a.length; i++) {
		a[i] = (100000.0 * a[i] * m) / 100000.0;
	    }
	}

	switch (g_mass_unit) {
	case "kg":
	    convert(this.x, 1/g_m_inch);
	    convert(this.y, 1/g_kg_lbs);
	    convert(this.y_land, 1/g_kg_lbs);
	    break;
	case "lb":
	    convert(this.x, g_m_inch);
	    convert(this.y, g_kg_lbs);
	    convert(this.y_land, g_kg_lbs);
	    break;
	default:
	    alert("Invalid lim_unit_change: cannot happen!");
	}

	this.min_arm  = Math.min.apply(Math, this.x);
	this.max_arm  = Math.max.apply(Math, this.x);
	this.min_mass = Math.min.apply(Math, this.y);
	this.max_mass = Math.max.apply(Math, this.y);
    }

    /* convert given moment arm value to the new range fitting the svg drawing area */

    function new_mom(old_value)
    {
	var old_range = g_lim.max_arm - g_lim.min_arm;
	var new_range = g_lim.x_draw_max - g_lim.x_draw_min;
	return ((old_value - g_lim.min_arm) * new_range)/old_range + g_lim.x_draw_min;
    }

    /* convert given mass value to the new range fitting the svg drawing area */

    function new_mass(old_value)
    {
	var old_range = g_lim.max_mass - g_lim.min_mass;
	var new_range = g_lim.y_draw_max - g_lim.y_draw_min;
	return ((old_value - g_lim.min_mass) * new_range)/old_range + g_lim.y_draw_min;
    }

    /* basic numeric value object with unit and type */

    function value_unit(v, u)
    {
	switch (u) {
	case "kg":
	case "lb":
	case "ltr":
	case "usg":
	case "min":
	case "qrt":
	    break;
	default:
	    // this should not happen
	    alert("value_unit with values: " + v + " " + u + " called.");
	}

	this.num  = v;
	this.unit = u;
    }

    /* convert minutes to hours */

    value_unit.prototype.time = function m2hrs()
    {
	var m, h;

	function pad2(n) {
	    return (n < 10?'0':'') + n;
	}
	
	h = this.num / 60.0;
	m = this.num % 60.0;
	
	return pad2(Math.floor(h)) + ":" + pad2(Math.floor(m)) + " h";
    }

    /* 
       display the value and unit
       n: num of decimals
    */
       
    value_unit.prototype.display = function v_display(n)
    {
	if (n) {
	    return this.num.toFixed(n) + " " + this.unit;
	}
	else {
	    return this.num + " " + this.unit;
	}
    }

    /* 
       return the SI value (used for saving defaults)
    */

    value_unit.prototype.get_si = function v_si()
    {
	var m = this.num;

	if (g_mass_unit == "kg") {
	    return m;
	}

	switch (this.unit) {
	case "usg":
	    m = (100000.0 * m * (1/g_ltr_usg)) / 100000.0;
	    break;
	case "lb":
	    m = (100000.0 * m * (1/g_kg_lbs)) / 100000.0;
	    break;
	case "qrt":
	    // TODO XXX implement this
	    break;
	case "ltr":
	case "kg":
	case "min":
	    // fall through
	    break;
	}
	return m;
    }

    /* 
       return the mass of value unit - calculates it for the volume units
       n: num of decimals
       u: also show unit
    */

    value_unit.prototype.getmass = function v_mass(n, u)
    {
	var m = this.num;
	
	switch (this.unit) {
	case "ltr":
	    if (g_mass_unit == "kg")
		m = (100000.0 * m * g_defs.ltr2kg) / 100000.0;
	    else
		m = (100000.0 * m * g_defs.ltr2kg * g_kg_lbs) / 100000.0;
	    break;
	case "usg":
	    if (g_mass_unit == "kg")
		m = (100000.0 * m * g_defs.usg2lb / g_kg_lbs) / 100000.0;
	    else
		m = (100000.0 * m * g_defs.usg2lb) / 100000.0;
	    break;
	case "qrt":
	    // TODO XXX implement this
	    break;
	default:
	    // fall through, display the raw value
	}
	
	// if no unit is wanted, try to return a number
	if (n) {
	    if (u)
		return m.toFixed(n) + (u?" " + g_mass_unit:"");
	    else
		return m.toFixed(n);
	}
	else {
	    if (u)
		return m + " " + g_mass_unit;
	    else
		return m;
	}
    }
    
    /* switch the unit and recalculate the value */
    
    value_unit.prototype.unitchg = function v_unit_change()
    {
	var that = this; // inner function trick
	var update_min_max = function(m, a) {
 	    if (that.lp) {
		that.lp.min = (100000.0 * that.lp.min * m) / 100000.0;
		that.lp.max = (100000.0 * that.lp.max * m) / 100000.0;
		that.lp.arm = (100000.0 * that.lp.arm * a) / 100000.0;
	    }
	}

	switch (this.unit) {
	case "kg":
	    this.unit  = "lb";
	    this.num = (100000.0 * this.num * g_kg_lbs) / 100000.0;
	    update_min_max(g_kg_lbs, g_m_inch);
	    break;
	case "lb":
	    this.unit = "kg";
	    this.num = (100000.0 * this.num / g_kg_lbs) / 100000.0;
	    update_min_max(1/g_kg_lbs, 1/g_m_inch);
	    break;
	case "ltr":
	    this.unit = "usg";
	    this.num = (100000.0 * this.num * g_ltr_usg) / 100000.0;
	    update_min_max(g_ltr_usg, g_m_inch);
	    break;
	case "usg":
	    this.unit = "ltr";
	    this.num = (100000.0 * this.num / g_ltr_usg) / 100000.0;
	    update_min_max(1/g_ltr_usg, 1/g_m_inch);
	    break;
	case "qrt":
	    // TODO XXX implement this
	    break;
	case "min":
	    // do nothing
	    break;
	default:
	    alert("Invalid v_unit_change: cannot happen!");
	}
    }

    /*
       l_point - a load point object

       id:     name of the global variable this is assigned to
       descr:  basic descriptive text for the item
       unit:   kg, ltr, min
       dflt:   mass of the load point (default value when constructed)
       min:    minimum value
       max:    max value (negative means do-not-print)
       arm:    moment arm
       step:   step size for input range element
    */

    function l_point(id, descr, unit, dflt, min, max, arm, step)
    {
	this.id      = id;
	this.descr   = descr;
	this.vu      = new value_unit(dflt, unit);
	this.min     = min;
	this.max     = Math.abs(max);
	this.dispmax = (max > 0 ?1:0);
	this.arm     = arm;
	this.step    = step;

	this.vu.lp   = this; // associate the mass object with this
    }

    /* 
      a table row with interactive elements

      lp:    load point object
     */

    function interactive_row(lp)
    {
	this.lp    = lp;
	this.id    = this.lp.id;
	this.celli = this.lp.id + "c1";  // interactive cell id
	this.cellr = this.lp.id + "c2";  // result cell id
	this.sldr  = this.lp.id + "s";   // slider element id
    }
    
    /* return the descriptive name for the row (first cell) */

    interactive_row.prototype.description = function i_descr()
    {
	var a = '<th>' + this.lp.descr[g_defs.s.lang];

	if (this.lp.dispmax) 
	    a += ' (max ' + this.lp.max.toFixed() + ' ' + this.lp.vu.unit +')';

	a += '</th>';
	return a;
    }

    /* return the interactive result value of this row (second cell) */

    interactive_row.prototype.iresult = function i_iresult()
    {
	var a = '<td style="background-color:yellow;" id="' + this.celli + '" ' +
	    'onclick="show_number_entry(\'' + this.celli + '\', \'' + this.sldr + '\')">';

	switch (this.lp.vu.unit) {
	case "min":
	    a += this.lp.vu.display();
	    break;
	default:
	    a += this.lp.vu.display(1);
	    break;
	}
	a += '</td>';
	return a;
    }

    /* return the result value of this row (last cell) */

    interactive_row.prototype.result = function i_result()
    {
	var a = '<td id="' + this.cellr + '">';

	if (this.lp.fuel_flow !== true) {

	    switch (this.lp.vu.unit) {
	    case "min":
		a += this.lp.vu.time();
		break;
	    default:
		a += this.lp.vu.getmass(1);
		break;
	    }
	}
	else {
	    // this is a fuel flow row
	    var b = new value_unit(this.lp.vu.num, this.lp.vu.unit);
	    b.unitchg();
	    a += b.display(1);
	}

	a += '</td>';
	return a;
    }

    /* generate content for the whole row */

    interactive_row.prototype.content = function i_content()
    {
	var a = this.description();

	a += this.iresult();

	a += '<th style="text-align:center;padding-right:0px;"> ';
	a +=  '<input id="' + this.sldr +'" ';
	a += 'type="range" ';
	a += 'min="' + this.lp.min + '" ';
	a += 'max="' + this.lp.max + '" value="' + this.lp.vu.num +'" ';
	if (this.lp.step)
	    a += 'step="' + this.lp.step +'" ';
	a += 'oninput="user_input_range(this,\'' + this.lp.id + '\')" ';
	a += 'onchange="user_input_range(this,\'' + this.lp.id + '\')"/></th>';
	a += this.result();
	
	return a;
    }

    /* actions to do when the units change */

    interactive_row.prototype.unitchg = function i_update()
    {
	var slider = document.getElementById(this.sldr);

	this.lp.vu.unitchg();
	slider.min = this.lp.min;
	slider.max = this.lp.max;
	slider.value = this.lp.vu.num;

	refresh(this);
    }


    /* 
      a table row with interactive checkbox elements

      lp:    load point object
     */

    function cbox_interactive_row(lp)
    {
	this.lp    = lp;
	this.id    = this.lp.id;
	this.celli = null;  // checkbox is here, can't be interactive cell id
	this.cellr = this.lp.id + "c2";  // result cell id
	this.cbox  = this.lp.id + "ck";  // checkbox element id
    }
    
    /* return the descriptive name for the row (first cell) */

    cbox_interactive_row.prototype.description = function i_cdescr()
    {
	var a = '<th>' + this.lp.descr[g_defs.s.lang] + '</th>';
	return a;
    }

    /* return the result value of this row (last cell) */

    cbox_interactive_row.prototype.result = function ic_result()
    {
	var a = '<td id="' + this.cellr + '">';
	a += this.lp.vu.getmass(1);
	a += '</td>';
	return a;
    }

    /* generate content for the whole row */

    cbox_interactive_row.prototype.content = function ic_content()
    {
	var chk = (this.lp.max == this.lp.vu.num?"checked":"");

	var a = this.description();

	a += '<td style="background-color:yellow;text-align:center;" id="' + this.celli + '">';
 	a += '<input id="' + this.cbox + '" type="checkbox" ' + chk + ' ' +
	    'onchange="user_checkbox(this, \'' + this.lp.id + '\', ' +
	    this.lp.min + ', ' + this.lp.max + ')"/>';

	a += '</td>';

	a += '<th style="font-weight:normal; text-align:center;">. . . . .</th>';

	a += this.result();
	
	return a;
    }

    /* actions to do when the units change */

    cbox_interactive_row.prototype.unitchg = function i_update()
    {
	this.lp.vu.unitchg();
	refresh(this);
    }

    /* 
       a table row with no interactive elements

       lp: the load point for this row
       op: special handling according to op
    */

    function non_interactive_row(lp, op)
    {
	this.lp    = lp;
	this.id    = this.lp.id;
	this.op    = op;
	this.lp.up = this; // callback for the load point
	this.celli = null; // no interactive cell here
	this.cellr = this.lp.id+"c1"; // result cell id
    }

    /* return the descriptive name for the row (first cell) */

    non_interactive_row.prototype.description = function ni_descr()
    {
	var a = '<th>'+ this.lp.descr[g_defs.s.lang];

	if (this.lp.dispmax)
	    a += " (max "+ this.lp.max.toFixed() + " " + this.lp.vu.unit + ")";

	a += '</th>';
	return a;
    }

    /* return the result value for this row */

    non_interactive_row.prototype.result = function ni_result()
    {
	var a = '<td id="' + this.cellr + '"';
	a += '>';

	switch (this.lp.vu.unit) {
	case "min":
	    a += this.lp.vu.time();
	    break;
	default:
	    a +=  this.lp.vu.getmass(1);
	    break;
	}

	a += '</td>';
	return a;
    }
    
    /* generate content for the whole row */

    non_interactive_row.prototype.content = function ni_content()
    {
	var a = this.description();

	switch (this.op) {
	case "reverse":
	    a += '<th style="font-weight:normal; text-align:center;">. . .</th>';
	    a += '<th style="font-weight:normal; text-align:center;">. . . . .</th>';
	    a += this.result();
	    break;
	default:
	    a += this.result();
	    a += '<th style="font-weight:normal; text-align:center;">. . . . .</th>';
	    a += '<th style="font-weight:normal; text-align:center;">. . .</th>';
	    break;
	}

	return a;
    }

    /* actions to do when the units change */

    non_interactive_row.prototype.unitchg = function ni_update()
    {
	this.lp.vu.unitchg();
	refresh(this);
    }

    /* refresh a row's content */

    function refresh(row)
    {
	document.getElementById(row.id).innerHTML = row.content();
    }

    /* access localstorage */

    function load_defaults()
    {
	if (typeof(Storage) !== "undefined") {
	    var i, t = null;

	    for (var n in g_defs.s) {
		i = localStorage.getItem("kaltsi.mb." + g_defs.acft + "." + n);
		if (i !== null) {
		    if (t === null) {
			t = setTimeout("load_reminder()", 100);
		    }

		    g_defs.s[n] = Number(i);
		}
	    }

	    /* do language setting chores */
	    refresh_selectors();
	    refresh_svg_texts();
	    document.getElementById("units_select").selectedIndex = g_defs.s.mass;
	    document.getElementById("lang_select").selectedIndex = g_defs.s.lang;
	    document.getElementById("settings_select").selectedIndex = g_defs.s.lang;
	}
    }

    /* save to localstorage */

    function save_settings()
    {
	if (typeof(Storage) !== "undefined") {
	    /* The order of saveables is important!

               Normalize the masses to kg before saving.
             */
	    var savevalues = [
		FRONT_SEAT.vu.get_si(),
BACK_SEAT.vu.get_si(),
FUEL.vu.get_si(),
BAGGAGE.vu.get_si(),
BAGGAGE2.vu.get_si(),
TAXI_FUEL.vu.get_si(),
FUEL_FLOW.vu.get_si(),
FLIGHT_TIME.vu.get_si(),

		(g_mass_unit=="kg"?0:1),
		g_defs.s.lang
	    ];
	    
	    var i = 0;
	    for (var n in g_defs.s) {
		localStorage.setItem("kaltsi.mb." + g_defs.acft + "." + n, Number(savevalues[i]));
		i++;
	    }

	    document.getElementById("reminder_cell").
	    textContent=(g_defs.s.lang == 0?
			 "Saved these settings":
			 "Asetukset tallennettu");
	    setTimeout("remove_reminder()", 3000);
	}
    }

    /* clear the localstorage */
    function clear_settings()
    {
	if (typeof(Storage) !== "undefined") {
	    for (var n in g_defs.s) {
		localStorage.removeItem("kaltsi.mb." + g_defs.acft + "." + n);
	    }
	    
	    document.getElementById("reminder_cell").
	    textContent=(g_defs.s.lang == 0?
			 "Cleared your default settings":
			 "Asetukset pyyhitty");
	    setTimeout("remove_reminder()", 3000);
	}
    }

    /* update the text on the first row */

    function refresh_title_row()
    {
	// correct language
 	document.getElementById("input_cell").textContent = 
	    (g_defs.s.lang == 0?"Input":"Sy??te");
	
	// set the unit name correctly
 	document.getElementById("mass_title").textContent = 
	    (g_defs.s.lang == 0?
	     "Mass":
	     "Massa") + 
	    " (" + g_mass_unit + ")";
    }

    /* initialize everything when the page is loaded */

    function setup()
    {
	var i = 0;

	// calculate the 1 usg fuel -> lb constant
	g_defs.usg2lb = (1 / g_ltr_usg) * g_kg_lbs * g_defs.ltr2kg;

	 

	// then load the default values for the sliders
	load_defaults();

	refresh_title_row();

	// update some static elements
	document.getElementById("doc_title").textContent=g_defs.acft + " M&B";
	document.getElementById("ac_type_hdr").textContent=g_defs.acft_type + " " + g_defs.acft;
	document.getElementById("ac_reg").textContent=g_defs.acft;
	document.getElementById("origin_text").textContent=g_defs.origin;

	var trs = 50;
	document.getElementById("translate_group").setAttribute('transform', 'translate(' + trs + ',-450)');

	/*
         All these are created global, since I'm not using 'var'.
         The variable name must be the same as the id.

        */

	rows = [ ];
	BASIC_WEIGHT=new l_point("BASIC_WEIGHT", ["Basic weight", "Perusmassa"], "kg", 697.10, 0, -1, 0.99);
rows.push(new non_interactive_row(BASIC_WEIGHT, "reverse"));
FRONT_SEAT=new l_point("FRONT_SEAT", ["Front seat", "Etupenkki"], "kg", g_defs.s.FRONT_SEAT, 25, -250, 0.940);
rows.push(new interactive_row(FRONT_SEAT));
BACK_SEAT=new l_point("BACK_SEAT", ["Back seat", "Takapenkki"], "kg", g_defs.s.BACK_SEAT, 0, -250, 1.854);
rows.push(new interactive_row(BACK_SEAT));
FUEL=new l_point("FUEL", ["Fuel", "Polttoaine"], "ltr", g_defs.s.FUEL, 0, 189, 1.219, 0.1);
rows.push(new interactive_row(FUEL));
BAGGAGE=new l_point("BAGGAGE", ["Baggage", "Matkatavaratila"], "kg", g_defs.s.BAGGAGE, 0, 54, 2.408);
rows.push(new interactive_row(BAGGAGE));
BAGGAGE2=new l_point("BAGGAGE2", ["Baggage 2", "Matkatavaratila 2"], "kg", g_defs.s.BAGGAGE2, 0, 22, 3.175);
rows.push(new interactive_row(BAGGAGE2));
TAXI_FUEL=new l_point("TAXI_FUEL", ["Taxi fuel", "Rullauspolttoaine"], "ltr", g_defs.s.TAXI_FUEL, 0, -20, 1.219, 0.1);
rows.push(new interactive_row(TAXI_FUEL));
TOW=new l_point("TOW", ["Take-off mass", "L??ht??massa"], "kg", 0, 0, 1043, 1);
rows.push(new non_interactive_row(TOW));
LNDW=new l_point("LNDW", ["Landing mass", "Laskumassa"], "kg", 0, 0, 1043, 1);
rows.push(new non_interactive_row(LNDW));
ENDURANCE=new l_point("ENDURANCE", ["Endurance", "Toiminta-aika"], "min", 0, 1, -800, 0);
rows.push(new non_interactive_row(ENDURANCE));
FUEL_FLOW=new l_point("FUEL_FLOW", ["Fuel flow/h", "Kulutus/h"], "ltr", g_defs.s.FUEL_FLOW, 5, -50, 1, 0.1); FUEL_FLOW.fuel_flow = true;
rows.push(new interactive_row(FUEL_FLOW));
FLIGHT_TIME=new l_point("FLIGHT_TIME", ["Flight time", "Lentoaika"], "min", g_defs.s.FLIGHT_TIME, 1, -300, 0);
rows.push(new interactive_row(FLIGHT_TIME));


	// This does not necessarily get a row, but acts just as a calculation object.
	g_lnd_fuel = new l_point("g_lnd_fuel", ["Landing fuel", "Laskupolttoaine"], "ltr", 0, 0, 0, 0);

	for (i = 0; i < rows.length; i++) {
	    var row = document.getElementById("table1").insertRow(-1);
	    row.id = rows[i].id;
	    row.innerHTML = rows[i].content();
	}

	// make sure the select element shows the first option
	document.getElementById("settings_select").selectedIndex = 0;

	draw_limits_svg();

	calculate();

	// if the mass has been updated from its default value of 0
	if (g_defs.s.mass == 1) {
	    var t = { value:"lb_usg" };
	    handle_unit_change(t);
	}
    }

    /* crunch the numbers */

    function calculate()
    {
	// take-off weight
	TOW.vu.num = calc_mass(
	    BASIC_WEIGHT,
	    FRONT_SEAT,
	    FUEL,
	    BAGGAGE,
	    BACK_SEAT,
BAGGAGE2,

	    null
	);
	TOW.vu.num -= TAXI_FUEL.vu.getmass();

	// landing fuel volume
	g_lnd_fuel.vu.num = FUEL.vu.num - (FUEL_FLOW.vu.num * FLIGHT_TIME.vu.num) / 60.0 - TAXI_FUEL.vu.num;

	// minimum landing fuel (45 mins reserve)
	g_lnd_fuel.min = (FUEL_FLOW.vu.num * 45.0) / 60.0;
	
	// landing weight
	LNDW.vu.num = calc_mass(
	    g_lnd_fuel,
	    FRONT_SEAT,
	    BAGGAGE,
	    BASIC_WEIGHT,
	    BACK_SEAT,
BAGGAGE2,

	    null
	);

	// endurance
	ENDURANCE.vu.num = ((FUEL.vu.num - TAXI_FUEL.vu.num) / FUEL_FLOW.vu.num) * 60.0; // minutes
	if (ENDURANCE.vu.num < 0)
	    ENDURANCE.vu.num = 0;
	    

	// compute the moment values for each load point
	calc_moment(BASIC_WEIGHT);
	calc_moment(FRONT_SEAT);
	calc_moment(BAGGAGE);
	calc_moment(TAXI_FUEL);
	calc_moment(FUEL);
	calc_moment(BACK_SEAT);
calc_moment(BAGGAGE2);


	// take-off moment arm
	TOW.moment = 
	    BASIC_WEIGHT.moment + 
	    FRONT_SEAT.moment + 
	    BAGGAGE.moment +
	    BACK_SEAT.moment +
BAGGAGE2.moment +

	    FUEL.moment - 
	    TAXI_FUEL.moment;

	TOW.arm = TOW.moment / TOW.vu.getmass();

	// landing moment arm
	LNDW.moment = 
	    BASIC_WEIGHT.moment + 
	    FRONT_SEAT.moment + 
	    BAGGAGE.moment + 
	    BACK_SEAT.moment +
BAGGAGE2.moment +

	    (g_lnd_fuel.vu.getmass() * FUEL.arm);

	LNDW.arm = LNDW.moment / LNDW.vu.getmass();

	debug = document.getElementById("debug");
	if (g_defs.debug) {
	    debug.textContent = BASIC_WEIGHT.moment.toFixed(3);
	    debug.textContent += (" " + FRONT_SEAT.moment.toFixed(3));
	    debug.textContent += (" " + BAGGAGE.moment.toFixed(3));
	    debug.textContent += (" " + TAXI_FUEL.moment.toFixed(3));
	    debug.textContent += (" " + FUEL.moment.toFixed(3));
	    debug.textContent += (" " + BACK_SEAT.moment.toFixed(3));
debug.textContent += (" " + BAGGAGE2.moment.toFixed(3));

	    debug.textContent += (" tow.arm=" + TOW.arm.toFixed(3));
	    debug.textContent += (" lnd.arm=" + LNDW.arm.toFixed(3));
	    debug.textContent += (" lnd fuel=" + g_lnd_fuel.vu.display(1));
	}
	else {
	    debug.textContent = "";
	}
	
	update_cells();
	update_svg();
    }

    /* show a reminder if the user has saved settings */

    function load_reminder()
    {
	document.getElementById("reminder_cell").
	textContent=(g_defs.s.lang == 0?
		     "Loaded default settings":
		     "Asetukset ladattu");
	
	setTimeout("remove_reminder()", 5000);
    }
    
    /* remove the settings reminder */
    
    function remove_reminder()
    {
	document.getElementById("reminder_cell").textContent="";
    }

    /* make sure the select elements use the correct language */

    function refresh_selectors()
    {
	var lang = {
	    "select_settings_title" : ["Settings",
				       "Asetukset" ],
	    "select_settings_save"  : ["Save",
				       "Tallenna" ],
	    "select_settings_clear" : ["Clear",
				       "Poista" ],
	    "units_kg_litre"        : ["kg & litre",
				       "kg & litra"]
	};

	var x = null;
	for (var n in lang) {
	    // update select elements
	    x = document.getElementById(n);
	    if (x !== null)
		x.text = lang[n][g_defs.s.lang];
	}
    }

    /* set the correct language */

    function refresh_svg_texts()
    {
	document.getElementById("takeoff_text").textContent=
	    (g_defs.s.lang == 0?"- Take-off":"- Lentoonl??ht??");

	document.getElementById("landing_text").textContent=
	    (g_defs.s.lang == 0?"- Landing":"- Lasku");
    }

    /* change all the language strings */
    
    function handle_lang_change(t)
    {
	switch (t.value) {
	case "Suomi":
	    g_defs.s.lang = 1;
	    break;
	case "English":
	    g_defs.s.lang = 0;
	    break;
	default:
	    break;
	}

	refresh_selectors();
	refresh_title_row();

	for (i = 0; i < rows.length; i++) {
	    refresh(rows[i]);
	}

	refresh_svg_texts();

	calculate();
    }
    
    /* change all numbers and their units */

    function handle_unit_change(t)
    {
	switch (t.value) {
	case "kg_litre":
	    g_mass_unit = "kg";
	    break;
	case "lb_usg":
	    g_mass_unit = "lb";
	    break;
	default:
	    return;
	    break;
	}

	refresh_title_row();

	// request unit change from all rows
	for (var i = 0; i < rows.length; i++) {
	    rows[i].unitchg();
	}

	// no row owns this object
	g_lnd_fuel.vu.unitchg();

	// update the limits
	g_lim.unitchg();

	calculate();
    }

    /*
       actions to do when user manipulates an input range element

       ir:  input range element reference
       lp:  global load point variable
    */

    function user_input_range(ir, lp)
    {
	eval(lp).vu.num = Number(ir.value);
	calculate();
    }

    /*
       user checkbox event
       cb:  checkbox reference
       lp:  global load point variable
       min: lp min value
       max: lp max value

       Seems that I can't use eval twice in this function. First I tried
       reading the min/max values using eval and then assigning them to
       the vu.num, but that fails. TODO: find out why.
     */

     function user_checkbox(cb, lp, min, max)
    {
	if (cb.checked == true)
	    eval(lp).vu.num = max;
	else
	    eval(lp).vu.num = min;

	calculate();
    }

    /* update the value cells in the table */

    function update_cells()
    {
	for (i = 0; i < rows.length; i++) {
	    if (rows[i].celli !== null) {
		document.getElementById(rows[i].celli).innerHTML = rows[i].iresult();
	    }
	    document.getElementById(rows[i].cellr).innerHTML = rows[i].result();
	}
    }

    /* take in an arbitrary number of value_unit objects and calculate their mass */

    function calc_mass(args)
    {
	var i, mass = 0;

	for (i = 0; i < arguments.length; i++) {
	    if (arguments[i] != null)
		mass += arguments[i].vu.getmass();
	}

	return mass;
    }

    /* calculate and add moment to given the object */

    function calc_moment(o)
    {
	o.moment = o.arm * o.vu.getmass();
    }

    /*
      Checks if given point (x,y) is inside the mass and balance envelope.
      http://www.ecse.rpi.edu/Homepages/wrf/Research/Short_Notes/pnpoly.html
    */

    function warn_limits(x, y, yarray)
    {
	var j = g_lim.x.length - 1;
	var in_envelope = 0;
	var i, yi, xi, yj, xj;
	
	for (i=0; i < g_lim.x.length; j=i++) {
	    xi = g_lim.x[i];
	    xj = g_lim.x[j];
	    yi = yarray[i];
	    yj = yarray[j];
	    
	    if ( ((yi > y) != (yj > y)))  {
		if (x < ((xj - xi) * (y - yi) / (yj - yi) + xi))
		    in_envelope = (in_envelope==1?0:1);
	    }
	}

	if (y < Math.max.apply(Math, yarray) && !in_envelope) {
	    // we're outside the x-boundary
	    in_envelope = -1;
	}
	
	return in_envelope;
    }
    

    /* draw the svg boxes and the line around the MnB envelope */

    function draw_limits_svg()
    {
	var polygon = "";

	var svg = document.getElementById("svg_main");
	svg.setAttribute('width',  600);
	svg.setAttribute('height', 480);
	
	var rct = document.getElementById("svg_border");
	rct.setAttribute('width',  550);
	rct.setAttribute('height', 450);

	for (var i = 0; i < g_lim.x.length; i++) {
	    polygon += new_mom(g_lim.x[i]) + ',' + new_mass(g_lim.y[i]) + " ";
	}
	
	document.getElementById("mb_limits").setAttribute('points', polygon);

	if (LNDW.max < TOW.max) {
	    var ll = document.getElementById("landing_limit");
	    ll.setAttribute('x1', new_mom(g_lim.lim_lnd1));
	    ll.setAttribute('y1', new_mass(LNDW.max));
	    ll.setAttribute('x2', new_mom(g_lim.lim_lnd2));
	    ll.setAttribute('y2', new_mass(LNDW.max));
	}

	update_svg();
    }

    /* draw the dynamic elements on the svg object */

    function update_svg()
    {
	var lx = new_mom(LNDW.arm);
	var ly = new_mass(LNDW.vu.getmass());
	var tx = new_mom(TOW.arm);
	var ty = new_mass(TOW.vu.getmass());

	var t = document.getElementById("takeoff_point");
	t.setAttribute('cy', ty);
	t.setAttribute('cx', tx);

	var c = document.getElementById("cg_arrow");
	c.setAttribute('x1', tx);
	c.setAttribute('y1', ty);
	c.setAttribute('x2', lx);
	c.setAttribute('y2', ly);

	var lang = {
	    "nofuel" : ["NOT ENOUGH LANDING FUEL",
		      "EI TARPEEKSI POLTTOAINETTA"],
	    "remain" : ["remaining", "j??ljell??"],
	    "mtow"   : ["TAKE-OFF OVERWEIGHT",
		      "YLIPAINOA LENTOONL??HD??SS??"],
	    "mlm"    : ["LANDING OVERWEIGHT",
		      "YLIPAINOA LASKUSSA"],
	    "cgbust" : ["CENTER OF GRAVITY OUT OF BOUNDS",
		      "MASSAKESKIPISTE RAJOJEN ULKOPUOLELLA"],
	    "ok"     : ["Limits OK", "Rajat OK"]
	};

	var warning = 1;
	var w = document.getElementById("warning_text");
	if (g_lnd_fuel.vu.num < g_lnd_fuel.min) {
	    w.setAttribute('fill', "red");
	    w.textContent = lang["nofuel"][g_defs.s.lang] + "! " +
		(g_lnd_fuel.vu.num < 0?"0":g_lnd_fuel.vu.display(1)) + " " +
		lang["remain"][g_defs.s.lang] + "!";
	}
	else if (warn_limits(TOW.arm, TOW.vu.getmass(), g_lim.y) == 0) {
	    w.setAttribute('fill', "red");
	    w.textContent = lang["mtow"][g_defs.s.lang] + " " +
		(TOW.vu.getmass() - TOW.max).toFixed(1) +
		" " + g_mass_unit + "!!";
	}
	else if (warn_limits(LNDW.arm, LNDW.vu.getmass(), g_lim.y_land) == 0) {
	    w.setAttribute('fill', "red");
	    w.textContent = lang["mlm"][g_defs.s.lang] + " " +
		(LNDW.vu.getmass() - LNDW.max).toFixed(1) + " " +
		g_mass_unit + "!!";
	}
	else if (warn_limits(TOW.arm, TOW.vu.getmass(), g_lim.y) < 0) {
	    w.setAttribute('fill', "red");
	    w.textContent = lang["cgbust"][g_defs.s.lang] + "!!";
	}
	else if (warn_limits(LNDW.arm, LNDW.vu.getmass(), g_lim.y) < 0) {
	    // This landing CG check uses the take-off weight limits
	    // (y-coords) because the MnB envelope is drawn according
	    // to the take-off values.
	    w.setAttribute('fill', "red");
	    w.textContent = lang["cgbust"][g_defs.s.lang] + "!!";
	}
	else {
	    w.setAttribute('fill', "green");
	    w.textContent = lang["ok"][g_defs.s.lang];

	    warning = 0;
	}

	function check_alert_cell(lp) {
	    if (lp.vu.getmass() > lp.max) {
		document.getElementById(lp.up.cellr).
		setAttribute('style', 'color:white; background-color:red;');
	    } else {
		document.getElementById(lp.up.cellr).
		setAttribute('style', 'background-color:none;');
	    }
	}

	if (warning == 1) {
	    document.getElementById("ac_reg").
	    setAttribute('style', 'opacity:0.25;fill:red;text-anchor:middle;');

	    document.getElementById("mb_limits").setAttribute('fill', 'yellow');

	    // recheck tow and lw here so that we can paint both cells
	    // if necessary
	    check_alert_cell(TOW);
	    check_alert_cell(LNDW);
	}
	else {
	    check_alert_cell(TOW);
	    check_alert_cell(LNDW);

	    document.getElementById("ac_reg").
	    setAttribute('style', 'opacity:0.15;fill:black;text-anchor:middle;');
	    document.getElementById("mb_limits").setAttribute('fill', 'white');
	}
    }

    /* global debug toggle handler */

    function toggle_debug()
    {
	g_defs.debug = !g_defs.debug;
	calculate();
    }

    /* user selects an option from settings */

    function handle_settings_change(t)
    {
	switch(t.value) {
	case "save":
	    save_settings();
	    break;
	case "clear":
	    clear_settings();
	    break;
	default:
	    return;
	    break;
	}

	// return the selector to default index
	t.selectedIndex = 0;
    }

    /* return the x and y coords for the given element */

    function getXYpos(elem)
    {
	if (!elem) {
	    return {"x":0,"y":0};
	}
	var xy={"x":elem.offsetLeft,"y":elem.offsetTop}
	var par=getXYpos(elem.offsetParent);
	for (var key in par) {
	    xy[key]+=par[key];
	}
	return xy;
    }

    function show_number_entry(id, slider)
    {
	var cell = document.getElementById(id);
	var cpos = getXYpos(cell);
	var x = cpos['x']-5;
	var y = cpos['y']-5;
	var input_div = document.getElementById("number");
	input_div.style.left = x + 'px';
	input_div.style.top = y + 'px';

	var input_num = document.getElementById("number_input");
	var s = document.getElementById(slider);
	var v = Number(s.value);
	if (s.step != '') {
	    v = v.toFixed(1);
	    input_num.step = s.step;
	} else {
	    v = v.toFixed(0);
	    input_num.step = '';
	}

	input_num.value = v;
	// abuse the input element's name attribute
	input_num.name = slider;

	input_div.style.display='block';

	// add an event handler to hide the div in case user touches
	// or clicks outside the number div in the table
	var tbl = document.getElementById("table1");
	tbl.onmouseover = function() {
	    var input_div = document.getElementById("number");
	    if (input_div.style.display === 'block') {
		input_div.style.display = 'none';
		tbl.onmouseover = null;
	    }
	}

	input_num.focus();
	input_num.select();

	/* this would pre-select the number in iPad when clicked, but
	 * does not work on desktop */
	//input_num.setSelectionRange(0, 9999);
    }

    function enter_number(slider, value)
    {
	if (value == '') return;

	var sldr = document.getElementById(slider);
	sldr.value = value;
	// hide the div
	document.getElementById("number").style.display = 'none';
	// unfocus the number input so that iPad keyboard goes away
	document.getElementById("number_input").blur();
	// remove the trigger to hide the div
	document.getElementById("table1").onmouseover = null;
	sldr.oninput();
    }

   // initialize everything when the document has been loaded
    window.onload = setup;

</script>

<title id="doc_title">Page title</title>

</head>
<body>
  
  <h1 id="ac_type_hdr">NO AIRCRAFT TYPE SET</h1>
  
  <select id="settings_select" onchange="handle_settings_change(this)">
    <option id="select_settings_title" value="title" disabled>Settings</option>
    <option id="select_settings_save"  value="save" >Save</option>
    <option id="select_settings_clear" value="clear">Clear</option>
  </select>

  <select id="lang_select" onchange="handle_lang_change(this)">
    <option value="English">English</option>
    <option value="Suomi">Suomi</option>
  </select>

  <button id="debug_button" type="button" title="Toggle debug info."
  onclick="toggle_debug()" style="display:none;">Debug</button>

  <p> </p>

  <table border="1" class="a" id="table1">
    <tbody>
      <tr>
	<th style="color:red;" id="reminder_cell"></th>
	<th style="width:75px;text-align:center;" id="input_cell">Input</th>
	<th style="text-align:center;width:175px;">
	  <select id="units_select" onchange="handle_unit_change(this)">
	    <option id="units_kg_litre" value="kg_litre" selected>kg & litre</option>
	    <option id="units_lb_usg" value="lb_usg">lbs & usg</option>
	  </select>
	</th>
	<th style="text-align:center;" id="mass_title">Mass</th>
      </tr>
    </tbody>
  </table>

  <!-- The number entry div -->
  <div id="number" style="display:none;" class="number_class">
    <input type="number" id="number_input" onchange="enter_number(this.name, this.value)" value="0">
  </div>

  <p id="debug"></p>

  <!-- SVG stuff here -->

  <svg id="svg_main" xmlns="http://www.w3.org/2000/svg" version="1.1" >
    <filter id = "my_shadow_filter" width = "150%" height = "150%">
      <feOffset result = "offOut" in = "SourceAlpha" dx = "2" dy = "2"/>
      <feGaussianBlur result = "blurOut" in = "offOut" stdDeviation = "3"/>
      <feBlend in = "SourceGraphic" in2 = "blurOut" mode = "normal"/>
    </filter>

    <defs>
      <path id="markerPath" d="M1,1 L1,7 L7,4 z"/>
    </defs>

    <g>
      <rect id="svg_border" x="5" y="10" width="1" height="1"
	    rx="20" ry="20" fill="white"
	    style="stroke:black;stroke-width:3;opacity:0.6;"
	    filter = "url(#my_shadow_filter)" />

      <g transform="scale(1,-1)">
	<g id="translate_group" >
	  <polygon id="mb_limits"
		   points="0,0 1,1"
		   fill="white"
		   style="stroke:black;stroke-width:2" />
	
	  <line id="landing_limit"
		x1="1" y1="1" x2="1" y2="1"
		style="fill:none;stroke:green;stroke-width:2" />

	  <marker id="markerArrow" markerWidth="8" markerHeight="8"
		  refx="4" refy="4" orient="auto">
	    <use xlink:href="#markerPath" style="fill: black;"/>
	  </marker>

	  <line id="cg_arrow" x1="3" y1="3" x2="4" y2="4"
		marker-end="url(#markerArrow)"
		style="stroke: black; stroke-width: 2px; stroke-dasharray: 6 3;"/>

	  <circle id="takeoff_point" cx="1" cy="1" r="5"
		  stroke="black" stroke-width="1" fill="red"/>
	
	</g>
      </g>

      <text id="ac_reg" x="50%" y="50%" font-size="80"
	    style="opacity:0.15;text-anchor:middle;">NO-REG</text>

      <!--g transform="translate(-50,325) rotate(-45)">
      <text id="ac_invalid" x="50%" y="50%" font-size="110"
	    style="opacity:0.5;text-anchor:middle;" fill="red"
	    visibility="/*INVALID_DATA*/hidden/**/">INVALID</text>
      </g-->

      <text id="warning_text" x="20" y="50" font-size="20"
	    fill="green" style="font-weight:bold;">Limits OK</text>

      <circle cx="25" cy="100" r="5" stroke="black" stroke-width="1" fill="red"/>
      <text x="35" y="105" id="takeoff_text">- Take-off</text>
    
      <!--circle cx="25" cy="125" r="5" stroke="black" stroke-width="1" fill="green"-->

      <g transform="translate(34,118) rotate(90) scale(2)">
	<use xlink:href="#markerPath"/>
      </g>
      <text x="35" y="130" id="landing_text">- Landing</text>
    </g>
  </svg>

  <footer>
    <hr/>
    
    <div id="extrainfo">
      <p class="aligncenter" id="origin_text">No origin</p>
      <p class="aligncenter"><small><a href="http://kaltsi.github.com/Mass-and-Balance/">m&amp;b sheets</a>  2016-04-14 15:18:42 +0300</small></p>
    </div>
    
  </footer>
  
</body>
</html>

