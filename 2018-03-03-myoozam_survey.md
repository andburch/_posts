---
layout:  post
title: "Where Do People Who Say 'Myoozam' Come From?"
comments:  true
published:  true
author: "Zach Burchill"
date: 2018-03-03 10:00:00
permalink: /myoozam_survey/
categories: [javascript,html5,sociolinguistics,museum,myoozam,"collecting data"]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



It was a nice evening, sitting around a fire under the stars with my new lab a few days after I started grad school. My girlfriend was visiting at the time, and I was gently teasing her about the way she pronounced the word "eggs" ("eygs").  

No one batted an eye.

It was then that she called me out for pronouncing "museum" as "myoo-zam."

**All hell broke loose.**

<!--more-->

## About me

I am just one of those superior human beings who pronounce the word "museum" in two syllables instead of three.[^1]

I have theories as to why I pronounce it like that (interpreting /eə/ before /m/ as the result of a raised /æ/ in Midwestern American English dialects), and I could explain why pronouncing it like that actually makes _more_ sense than the way most people do it, but my lab was having none of it. My pronunciation of that word quickly became a meme that follows me to this very day. 

## Where does it come from?

Part of why I'm teased so much about it is that this particular linguistic affectation doesn't seem to a documented to any region or dialect. In fact, half of my family (including my twin brother) doesn't even pronounce it this way. I could write this quirk off as a one-time "glitch" in human language learning (which happen all the time) but for the fact that my labmates and I have run into _other_ /myoo-zam/ speakers every once in a while. 

I have no idea whether (like the X-Men we truly are) us "myoozamers" have popped up _spontaneously_ across the country, or if there is some sort of region where this happens in more. I have scant information that suggests this could be something in western Pennsylvania and Texas, but the data is honestly pretty anecdotal.

## Help me!

I'm trying to get a sense of where this phenomenon is coming from (as well as learning more about Google API--more blog posts on that coming soon), so I've put together a survey to collect some basic information.

Please share this with anyone you know who pronounces "museum" as /myoo-zam/ or put it on Twitter so other people can pass it on. If you yourself have this quirk, please take the survey so we can understand what the heck is happening.[^2] If you have any information you think might be important to solving this mystery, leave a comment below or hit me up on Twitter ([@zachburchill](https://twitter.com/zachburchill))!

If you're interested in collecting this type of data yourself, I'll be releasing a little tutorial about how I made this survey, Google Map interface, and collection process in a blog post in a little bit, so stay tuned! 

<form class="form" name="submit-to-google-sheet" action="">
  <h2>Where Does "Myoozam" Come From?</h2>
  <p><label for="name">Name:</label><input placeholder="First name, full name, fake name, whatever" id="name" name="name">
  </p>
  <p><label for="myuzam">Do you pronounce "museum" as <span style="white-space: nowrap;">/myoo-zam/</span>? (two syllables)</label>
    <span style="width:100%;display:block;margin-top:7px;">
  <input type="radio" name="myoozam" id="myoozam_yes" value="yes" required>  &nbsp;<label for="myoozam_yes"> Yes </label> &nbsp; &nbsp; &nbsp;
  <input type="radio" name="myoozam" id="myoozam_no" value="no" required> &nbsp;<label for="myoozam_no"> No </label> 
  </span>
  </p>

  <p><label for="two_syll_hearsay">Do you hear/say "museum" as two syllables?</label>
    <select id="two_syll_hearsay" name="two_syll_hearsay" required>
      <option disabled selected value> -- select one -- </option>
      <option value="hear">I HEAR it as two syllables</option>
      <option value="say">I SAY it as two syllables</option>
      <option value="both">I both hear and say it as two syllables</option>
      <option value="neither">I neither hear nor say it as two syllables</option>
    </select>
  </p>
  <p><label>Where did you grow up?</label></p>
  <p><b>Add</b> a marker and <b>drag</b> it to any region you lived <b>for more than five years</b> growing up (until you were 14) on the map below. If you grew grew up in multiple regions, add them in order of how long you lived there, <b>starting with the place you lived longest</b>.</p>
  <p>You don't need to be super precise&mdash;unless you moved over 100 miles, don't count them as "different" locations!
  </p>
  <p>
    <div style="justify-content:center;flex-wrap:wrap;display:flex;margin-bottom:12px;">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="addPointer(you_map_array_obj, you_map);" type="button" value="Add location">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="removeLastPointer(you_map_array_obj, you_map);" type="button" value="Remove last location">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="removeAllPointers(you_map_array_obj, you_map);" type="button" value="Remove all locations">
    </div>
    <div class="map_holder_holder" style="width:100%;max-height:300px">
      <div class="map_holder" id="you_map"></div>
    </div>
  </p>

  <p><label>Where did your parents grow up?</label></p>
  <p style="text-align:justify;">The people who raise us exert a strong influence on the way we speak, so it's important to look at their language backgrounds too! Do the same thing as above, but mark where your parents or guardians grew up. <b>It doesn't have to be exact</b>&mdash;just
    complete it to the best of your knowledge. </p>
  <p>If you were only raised by one parent, or don't know their backgrounds at all, feel free to leave these maps blank.
  </p>
  <p><label>Parent A:</label></p>

  <p>
    <div style="justify-content:center;flex-wrap:wrap;display:flex;margin-bottom:12px;">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="addPointer(parent_a_map_array_obj, parent_a_map);" type="button" value="Add location">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="removeLastPointer(parent_a_map_array_obj, parent_a_map);" type="button" value="Remove last location">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="removeAllPointers(parent_a_map_array_obj, parent_a_map);" type="button" value="Remove all locations">
    </div>
    <div class="map_holder_holder" style="width:100%;max-height:300px">
      <div class="map_holder" id="parent_a_map"></div>
    </div>
  </p>

  <p><label>Parent B:</label></p>
  <p>
    <div style="justify-content:center;flex-wrap:wrap;display:flex;margin-bottom:12px;">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="addPointer(parent_b_map_array_obj, parent_b_map);" type="button" value="Add location">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="removeLastPointer(parent_b_map_array_obj, parent_b_map);" type="button" value="Remove last location">
      <input style="width:auto;margin:3px;0;0;3px;" onclick="removeAllPointers(parent_b_map_array_obj, parent_b_map);" type="button" value="Remove all locations">
    </div>
    <div class="map_holder_holder" style="width:100%;max-height:300px">
      <div class="map_holder" id="parent_b_map"></div>
    </div>
  </p>

  <p><label for="parents_say_darnedest">Have you ever heard your parents say <span style="white-space: nowrap;">/myoo-zam/</span>?</label>
    <select id="parents_say_darnedest" name="parents_say_darnedest" required>
      <option disabled selected value> -- select the best -- </option>
      <option value="parent_a">I've heard Parent A say it.</option>
      <option value="parent_b">I've heard Parent B say it.</option>
      <option value="both">I've heard both say it.</option>
      <option value="neither">I've never heard either say it.</option>
      <option value="idk">I don't remember/can't recall.</option>
    </select>
  </p>

  <p><label for="comments">Additional comments/information</label>
    <textarea style="resize:none;" id="comments" name="comments" placeholder="Anything else you think might be important to know?"></textarea>
  </p>

  <p><label for="wtf">Do you want this information to be taken seriously?</label>
    <span style="width:100%;display:block;margin-top:7px;">
      <input type="radio" name="kidding" id="not_kidding" value="good" required> <label for="not_kidding"> Yes </label> &nbsp; 
      <input type="radio" name="kidding" id="kidding" value="no" required> <label for="kidding"> No </label> &nbsp; 
      <input type="radio" name="kidding" id="zachfriend" value="zachfriend" required><label for="zachfriend"> No, I'm one of Zach's friends and I'm just messing with him </label> 
    </span>
  </p>
  <input type="hidden" id="you_positions" name="you_positions" value="">
  <input type="hidden" id="parent_a_positions" name="parent_a_positions" value="test">
  <input type="hidden" id="parent_b_positions" name="parent_b_positions" value="test">
  <p>
    <button type="submit">Send Message</button>
  </p>
</form>

<script>
const scriptURL = 'https://script.google.com/macros/s/AKfycbwoM7O8LsMnGW5L9YdpZOXiBMM01Jy0flixRV-9tJ4CZQsFLNrm/exec'
const form = document.forms['submit-to-google-sheet']
var map;
var you_map_array_obj = { l: [] };
var parent_a_map_array_obj = { l: [] };
var parent_b_map_array_obj = { l: [] };
function initMap() {
  you_map = new google.maps.Map(document.getElementById("you_map"), {
    center: { lat: 43.1, lng: -77.5 },
    zoom: 4,
    mapTypeControl: false,
    streetViewControl: false,
    rotateControl: false,
    fullscreenControl: false
  });

  parent_a_map = new google.maps.Map(document.getElementById("parent_a_map"), {
    center: { lat: 43.1, lng: -77.5 },
    zoom: 4,
    mapTypeControl: false,
    streetViewControl: false,
    rotateControl: false,
    fullscreenControl: false
  });

  parent_b_map = new google.maps.Map(document.getElementById("parent_b_map"), {
    center: { lat: 43.1, lng: -77.5 },
    zoom: 4,
    mapTypeControl: false,
    streetViewControl: false,
    rotateControl: false,
    fullscreenControl: false
  });
}

function addPointer(marker_array_obj, map_var) {
  var l = marker_array_obj.l.length + 1;
  marker_array_obj.l.push(makePointer(map_var, l.toString()));
}

function removeLastPointer(marker_array_obj, map_var) {
  var removing_marker = marker_array_obj.l.pop();
  removing_marker.setMap(null);
}

function removeAllPointers(marker_array_obj, map_var) {
  for (var i = 0; i < marker_array_obj.l.length; i++) {
    marker_array_obj.l[i].setMap(null);
  }
  marker_array_obj.l = [];
}

function makePointer(map_var, label_var) {
  var marker = new google.maps.Marker({
    position: map_var.getCenter(),
    label: label_var,
    zoom: 5,
    map: map_var,
    draggable: true,
    title: "Drag me!"
  });
  return marker;
}

function closeUpShop() {
  if (you_map_array_obj.l.length === 0) {
    alert("You must mark where you grew up to submit");
    return false;
  } else {
    document.getElementById("you_positions").value = getPositions(you_map_array_obj);
    document.getElementById("parent_a_positions").value = getPositions(parent_a_map_array_obj);
    document.getElementById("parent_b_positions").value = getPositions(parent_b_map_array_obj);
    return true;
  }
}

function getPositions(marker_array_obj) {
  var new_list = [];
  for (var i = 0; i < marker_array_obj.l.length; i++) {
    var pos = marker_array_obj.l[i].getPosition();
    var lat = pos.lat().toFixed(3);
    var lng = pos.lng().toFixed(3);
    new_list.push([lat,lng].join(","));
  }
  if (new_list.length === 0) { return "undefined";} 
  else {return new_list.join(",");}
}

form.addEventListener('submit', e => {
	if (closeUpShop() == true) {
      e.preventDefault()
      fetch(scriptURL, { method: 'POST', body: new FormData(form)})
        .then(response => console.log('Success!', response))
        .catch(error => console.error('Error!', error.message))
      alert("Thanks for your submission!");
      window.open("{{ site.url }}{{ site.baseurl }}thank_you","_self")
    }
  })
  
</script>
<script src="https://maps.googleapis.com/maps/api/js?key=AIzaSyDWpSc7HqHCjw-KWkb_H5ae8vpZNjtYkZA&callback=initMap" async defer></script>

<hr />
<br />

## Source Code:

I'll be releasing another blog post about my experience with Google Maps JavaScript API and how I'm saving the results of these surveys automatically to a Google Sheets document. Stay tuned for the code! 

### Footnotes

[^1]: Just to be clear, I'm being tongue-in-cheek here. Linguists know that no dialect or way of speaking is "superior" to any other. <br /><br />With the exception of their own, of course.

[^2]: This is not a real "study" (in any IRB sense)---it's just me wanting to sate my own personal curiosity. However, the name you put down, which is optional, will never be released without your consent, and neither will your parent(s)' locations ever be connected with yours (which could conceivably help people identify the data points?). Not that any of that really matters, but still.
