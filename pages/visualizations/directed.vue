<template>
  <HeaderViz />
  <Tooltip 
    :description="popupDescription"
    :title="popupTitle"
  />
  <BaseContent 
    title="Force-Directed Graph of Author Meetings and Material Exchanges"
    part1="
    If you select an author from the drop-down menu below, it will generate a graph of all their meetings and material exchanges with other LGBTQ+ exile writers organized according to the relative density of connections. Material exchanges indicate instances where one LGBTQ+ exile writer read, reviewed, or alluded to another writer and their work or corresponded with them. 
    "
    part2="
    The relative density of connections between writers is indicated by the thickness of the lines in the graph, with thick lines indicating a higher number of connections and thin lines a lower number. You can choose to select meetings or material exchanges in the check boxes above the graph. If you click on a particular connection between writers on the graph, a pop-up box will give you further information about the meetings or creative exchanges that occurred between them. 

    "
  />

  <BaseLoader :displayLoader="displayLoader" padding="py-44" />

  <div v-if="displayLoader != 'block' ">
    <div class="py-12">
      <FormKit
        v-model="author"
        type="select"
        label="Select an Author"
        name="author"
        inner-class="ring-0 focus-within:ring-gray-500"
        :options="allAuthors"
      />

      <AuthorCard 
        :authorInfo="authorInfo"
      />

      <FormKit
        v-model="selectedEvents"
        type="checkbox"
        label=" "
        :options="['Meetings', 'Material Exchanges']"
        help=" "
        outer-class="mt-12 !mb-0"
        fieldset-class="border-0"
        options-class="flex gap-8"
        wrapper-class="!items-end border-0"
        decorator-class="text-gray-500 peer-checked:ring-gray-500 peer-checked:text-gray-500"
      />

      <div class="">
        <div class="p-2 flex justify-end gap-4">
          <div class="flex items-end">
            <svg class="w-4 h-4 inline-block">
              <circle cx="6" cy="6" r="6" fill="var(--beige)" />
            </svg>
            <span class="text-beige text-sm font-bold">Meetings</span>
          </div>
          <div>
            <svg class="w-4 h-4 inline-block">
              <circle cx="6" cy="6" r="6" fill="var(--dark-brown)" />
            </svg>
            <span class="text-dark-brown text-sm font-bold">Material Exchanges</span>
          </div>
        </div>
      </div>

    </div>

    <div id="directed">
    </div>
    
    <div class="py-24">
    </div>
  </div>


</template>

<script setup>
import * as d3 from 'd3'

/* LOADER */
const allAuthors = useState('allAuthors')
const displayLoader = ref('block')
watch(() => allAuthors.value, (newValue, oldValue) => {
  displayLoader.value = 'none';
})
onMounted(() => {
  if (allAuthors.value.length > 0) {
    displayLoader.value = 'none';
  }
})

const selectedEvents = ref(['Meetings', 'Material Exchanges'])

// https://observablehq.com/@namitamo/force-directed-graph

const author = ref('')
const authorInfo = ref('')

const peopleDataset = useState('peopleDataset')
const directedMeetings = useState('directedMeetings')
const directedMaterial = useState('directedMaterial')

const popupDescription = ref('')
const popupTitle = ref('')
const popupVisible = useState('popupVisible', () => false)

watch(() => [author.value, selectedEvents.value], (newValue, oldValue) => {

  if (selectedEvents.value.length == 0) {
    // svg cleaning
    d3.select("#directed").selectAll("*").remove();
  } else {

    if ( author.value != '' ) {
      const vals = peopleDataset.value.filter(x =>  x.ID_1 == author.value)[0]
      authorInfo.value = {
        name: vals['ID_1'],
        dob: vals['Date of birth'],
        dod: vals['Date of death'], 
        occupation: vals['Occupation'],
        orientation: vals['Sexual orientation']
      }

      function prepareDataset(datasetArray) {
        let nodesArray = []
        let linksArray = []

        // nodes creation
        for (let i = 0; i<datasetArray.length; i++) {
          nodesArray.push([])
          let filteredDataset = datasetArray[i].filter(e => {
            return e.participants.includes(author.value);
          });

          filteredDataset.forEach((e, idx) => {
            for (let j = 0; j<e.participants.length;j++) {
              nodesArray[i].push({
                "id": e.participants[j],
                "group": e.participants[j] === author.value ? 1 : 2,
                "serie": i
              })
            }
          })
          nodesArray[i] = [...new Map(nodesArray[i].map(v => [JSON.stringify([v.id,v.group,v.serie]), v])).values()]
        }

        for (let i = 0; i<datasetArray.length; i++) {
          nodesArray[i].forEach((e, idx) => {
  
            let filteredDataset = datasetArray[i].filter(e => {
              return e.participants.includes(author.value);
            });

            for (let j=0;j<filteredDataset.length; j++) {
              for (let k=0; k<filteredDataset[j].participants.length;k++) {

                if (filteredDataset[j].participants.includes(e.id)) {
                  if (e.id != filteredDataset[j].participants[k]) {
                    if (linksArray.filter(it => it.serie === i && it.source === e.id && it.target === filteredDataset[j].participants[k]).length > 0) {
                      linksArray.filter(it => it.serie === i && it.source === e.id && it.target === filteredDataset[j].participants[k])[0].value += 1;
                      linksArray.filter(it => it.serie === i && it.source === e.id && it.target === filteredDataset[j].participants[k])[0].meetings.push({
                        "dateStart": filteredDataset[j].dateStart,
                        "dateEnd": filteredDataset[j].dateEnd,
                        "notes": filteredDataset[j].notes,
                        "source": filteredDataset[j].source,
                        "page": filteredDataset[j].page,
                        "participants": filteredDataset[j].participants,
                        "dob": filteredDataset[j].dob,
                        "dod": filteredDataset[j].dod,
                        "sexuality": filteredDataset[j].sexuality,
                        "job": filteredDataset[j].job,
                        "nationality": filteredDataset[j].nationality,
                        "type": filteredDataset[j].type
                      })
                    } else {
                      linksArray.push({
                        "source": e.id,
                        "target": filteredDataset[j].participants[k],
                        "value": 1,
                        "serie": i,
                        "meetings": [{
                          "dateStart": filteredDataset[j].dateStart,
                          "dateEnd": filteredDataset[j].dateEnd,
                          "notes": filteredDataset[j].notes,
                          "source": filteredDataset[j].source,
                          "page": filteredDataset[j].page,
                          "participants": filteredDataset[j].participants,
                          "dob": filteredDataset[j].dob,
                          "dod": filteredDataset[j].dod,
                          "sexuality": filteredDataset[j].sexuality,
                          "job": filteredDataset[j].job,
                          "nationality": filteredDataset[j].nationality,
                        "type": filteredDataset[j].type
                        }]
                      })
                    }
                  }
                }
              }
            }
          })

        }

        // remove duplicates
        linksArray = [...new Map(linksArray.map(v => [JSON.stringify([v.source,v.target,v.serie]), v])).values()]

        // clean links
        linksArray.forEach((e, idx) => {
          if (e.source > e.target && e.serie == 0) {
            const newSource = e.target;
            const newTarget = e.source;
            e.source = newSource;
            e.target = newTarget;
          } else if (e.source < e.target && e.serie == 1) {
            const newSource = e.target;
            const newTarget = e.source;
            e.source = newSource;
            e.target = newTarget;
          }
        })

        // clean nodes
        nodesArray = nodesArray.flat()
        nodesArray = [...new Map(nodesArray.map(v => [JSON.stringify([v.id,v.group]), v])).values()]

        return {
          "nodes": nodesArray,
          "links": linksArray
        }
      }

      const datasets = [];
      if (selectedEvents.value.includes('Meetings')) {
        datasets.push(directedMeetings.value)
      }
      if (selectedEvents.value.includes('Material Exchanges')) {
        datasets.push(directedMaterial.value)
      }
      let data =  prepareDataset(datasets)
      
      data.links.sort((a, b) => {
        if(a.value < b.value) { return -1; }
        if(a.value > b.value) { return 1; }
      })
      

      // dynamic height 
      let height;

      if (data.nodes.length < 4) {
        height = 300
      } else if (data.nodes.length < 8) {
        height = 500
      } else {
        height = 1000
      }

      const width = 1100;

      // svg cleaning
      d3.select("#directed").selectAll("*").remove();

      if (data.links.length > 0) {

        // The force simulation mutates links and nodes, so create a copy
        // so that re-evaluating this cell produces the same result.
        const links = data.links.map(d => ({...d}));
        let nodes = data.nodes.map(d => ({...d}));

        // let simulation;

        // Create a simulation with several forces.
        const simulation = d3.forceSimulation(nodes)
            /*
            .force("charge", d3.forceManyBody().strength(-2000))
            */
            .force("link", d3.forceLink(links).id(d => d.id))
            .force("charge", d3.forceManyBody().strength(-2000))
            .force('collision', d3.forceCollide().radius(70))
            .force("center", d3.forceCenter(width / 2, height/2))
            .on("tick", ticked);

        // Create the SVG container.
        const svg = d3.select("#directed")
            .append("svg")
            .attr("width", width)
            .attr("height", height)
            .attr("viewBox", [0, 0, width, height])
            .attr("style", "max-width: 100%; height: auto;");

        let link;

        if (selectedEvents.value.length > 1) {

          let linkColor = ['#d9d1b8','#401f13']

          /*
          link = svg.append("g")
            .selectAll()
            .data(links)
            .join("line")
              .attr("stroke", d => d.serie == 0 ? linkColor[0] : linkColor[1])
              .attr("stroke-opacity", 0.8)
              .attr("class", "paths cursor-pointer")
              .attr("stroke-width", d => Math.max(1.5, Math.sqrt(d.value * 6)))
              .on("click", click)
              .on("mouseout", mouseout)
              .on("mouseover", mouseover);
              */


              
          link = svg.append("g")
            .selectAll("path")
            .data(links)
            .join("path")
              .attr("stroke-dasharray", 0)
              .attr("stroke-opacity", 0.8)
              .attr("class", "paths cursor-pointer")
              .attr("stroke", d => d.serie == 0 ? linkColor[0] : linkColor[1])
              .attr("stroke-width", d => Math.max(1.5, Math.sqrt(d.value * 6)))
              .attr("fill", "transparent")
              .on("click", click)
              .on("mouseout", mouseout)
              .on("mouseover", mouseover);
              
        } else {

          let linkColor = '#d9d1b8';

          if (selectedEvents.value[0] == 'Material Exchanges') {
            linkColor = '#401f13'
          }

          link = svg.append("g")
            .selectAll()
            .data(links)
            .join("line")
              .attr("stroke", linkColor)
              .attr("stroke-opacity", 0.8)
              .attr("class", "paths cursor-pointer")
              .attr("stroke-width", d => Math.max(1.5, Math.sqrt(d.value * 6)))
              .on("click", click)
              .on("mouseout", mouseout)
              .on("mouseover", mouseover);
        }

        const node = svg.append("g")
            .attr("stroke", "#000")
            .attr("stroke-width", 1.5)
          .selectAll("g")
          .data(nodes)
          .join("g")

        const circle = node.append("circle")
            .attr("stroke", d => d.group != 1 ? "#000" : "#fff")
            .attr("stroke-width", d => d.group == 1 ? 3 : 1.5)
            .attr("r", 10)
            .attr("fill", d => d.group == 1 ? "#000" : "#fff"); // '#d9d1b8', '#401f13', '#736355'

        const text = node.append("text")
          .attr('class', 'text-stroke')
          .attr('text-anchor', 'middle')
          .attr('alignment-baseline', 'middle')
          .attr('dy', -15)
          .text(d => d.id)
        
        node.append("title")
          .text(d => d.id);

        // Set the position attributes of links and nodes each time the simulation ticks.
        function ticked() {
          if (selectedEvents.value.length > 1) {
          // if (selectedEvents.value.length > 1) {
            // creating the arcs
            link
              .attr("d", function(d) {
                const dx = d.target.x - d.source.x,
                    dy = d.target.y - d.source.y,
                    dr = Math.sqrt(dx * dx * 2.9 + dy * dy * 2.9);
                return "M" + d.source.x + "," + d.source.y + "A" + dr + "," + dr + " 0 0,1 " + d.target.x + "," + d.target.y;
              })
          } else {
            link
              .attr("x1", d => d.source.x)
              .attr("y1", d => d.source.y)
              .attr("x2", d => d.target.x)
              .attr("y2", d => d.target.y);
          }

          text
            .attr("cx", d => d.x)
            .attr("cy", d => d.y);

          node.attr('transform', d => `translate(${d.x},${d.y})`)
        }

        function mouseover(event) {
          d3.selectAll(".paths").attr("stroke-opacity", 0.1)
          d3.select(event.target).attr("stroke-opacity", 1)
        }

        function mouseout() {
          d3.selectAll(".paths").attr("stroke-opacity", 0.8)
        }

        function click(event, el) {

          let persons = []

          if (el.source.id != author.value) {
            persons.push(el.source.id)
          }
          if (el.target.id != author.value) {
            persons.push(el.target.id)
          }

          const city = el.target.name
          const evs = el.meetings

          // sort material exchange by type in alphabetical order
          
          
          if (evs[0].type != undefined) {
            evs.sort((a, b) => {
              if(a.type < b.type) { return -1; }
              if(a.type > b.type) { return 1; }
            })
          }

          let evenement = ''
          if (el.serie == 0 && selectedEvents.value[0] == 'Meetings') {
            evenement = 'Meeting'
          } else {
            evenement = 'Material Exchange'
          }

          if (evs.length > 1) {
            evenement += 's'
          }

          popupTitle.value = `
            ${evs.length} ${evenement} involving ${persons.join(', ')}
          `

          popupDescription.value = ''



          for ( let i=0 ; i<evs.length; i++) {

            console.log("evs", evs[i])


            let page = evs[i].page;
            if (page.includes('-')) {
              page = 'pp. ' + page
            } else {
              page = 'p. ' + page
            }

            let matExchangeType = '';
            if (evs[i].type !== undefined) {
              matExchangeType += `
                ${evs[i].type}<br>
              `
            }

            let participants = [];
            let participantString = '<br>';
            for (let j = 0; j<evs[i].participants.length; j++) {
              participants.push({
                "name": evs[i].participants[j],
                "dob": evs[i].dob[j],
                "dod": evs[i].dod[j],
                "nationality": evs[i].nationality[j],
                "sexuality": evs[i].sexuality[j],
                "job": evs[i].job[j]
              })

              let article = 'a';
              let vowelTest = '^[aieouAIEOU].*';
              if (participants[j].nationality.join('/').split('')[0].match(vowelTest)) {
                article = 'an'
              }

              participantString += `
                ● ${participants[j].name} (${participants[j].dob} — ${participants[j].dod}) was ${article} 
                ${participants[j].nationality.join('/')}, ${participants[j].sexuality}, ${participants[j].job.join(', ')}.
                <br>
              `
            }

            participantString += '<br>'

            let notesDescription = '';
            // title of books in italic
            if (evs[i].notes != 'undefined') {
              notesDescription = ": " + evs[i].notes
              notesDescription = notesDescription.split('_')
              const notesFormatted = []
              for (let i = 0; i<notesDescription.length; i++) {
                notesFormatted.push(notesDescription[i])
                if (i%2==0 && i<notesDescription.length-1) {
                  notesFormatted.push('<i>')
                } else if (i%2==1) {
                  notesFormatted.push('</i>')
                }
              }
              notesDescription = notesFormatted.join('');
            };
            
            popupDescription.value += `
              <div class="pb-2 my-2 border-b border-dashed border-gray-500 text-sm">
                <span class='underline'>${matExchangeType}</span>
                <div class="py-2">
                  <span class='font-sans font-bold'>${evs[i].dateStart} — ${evs[i].dateEnd}: </span>
                  <span class=''>${notesDescription}</span>
                </div>
                <div><span class='font-sans font-bold'>Participants: </span>
                  ${participantString}
                </div>
                <div>

                  <span class='font-sans font-bold'>Source: </span>
                  <span class='italic'>${evs[i].source.split('_')[0].split('by')[0]}</span> by <span>${evs[i].source.split('_')[0].split('by')[1]}</span>
                  (${page})
                </div>
              </div>
            `
          }

          popupVisible.value = true
        }

      } else {
        const message = `
          <div>No meetings/material exchanges registered for ${author.value}</div>
        `
        d3.select("#directed").html(message)
      }
    }
  }

})



</script>

<style>
.text-stroke {
  font-size:  18px;
  paint-order: stroke;
  stroke: #fff;
  fill: #000;
  stroke-width: 2px;
  stroke-linecap: butt;
  stroke-linejoin: miter;
  font-weight: 800;
}

#directed {
  width:1100px !important;
}
</style>