<script>
  import { onMount } from "svelte";
  import { handleFiles } from "./imageupload.js";
  import { post_www_url_encoded } from "./sendcommmand.js";

  const LEDPanelCount = 3072;
  let LEDPanelData = new Array(LEDPanelCount).fill(0);
  const LEDCount = 256;
  const hiddencheckboxclassName = Array.from(
    Array(LEDCount).keys(),
    (x) => "c" + x
  );
  const hiddenlabelclassName = Array.from(Array(LEDCount).keys(), (x) => [
    "s" + x,
    "c" + x,
  ]);
  const panels = Array.from(Array(12).keys(), (x) => x + 1);
  let current_panel = 1;
  const panel_width = 96;

  function str2ab(str) {
    var buf = new ArrayBuffer(str.length); // 2 bytes for each char
    var bufView = new Uint8Array(buf);
    for (var i = 0, strLen = str.length; i < strLen; i++) {
      bufView[i] = str.charCodeAt(i);
    }
    return buf;
  }

  function binaryToByteArray() {
    const pbmData = new Uint8Array(Math.floor(LEDPanelCount / 8));
    let index;
    let val = 0;
    let pbmIndex = 0;
    let endOfByte;
    for (let i = 0; i < LEDPanelCount; i++) {
      val = val | (LEDPanelData[i] << (7 - (i % 8)));
      endOfByte = (i + 1) % 8;
      if (endOfByte === 0) {
        pbmData[pbmIndex] = val;
        val = 0;
        pbmIndex = pbmIndex + 1;
      }
    }
    return pbmData;
  }

  function SaveFile() {
    //const data = new Uint8Array(LEDPanelData);
    const data = binaryToByteArray();
    const headerstring = "P4\n96 32\n";
    //const header = new Uint8Array(9);

    const header = str2ab(headerstring);
    const headerView = new Uint8Array(header);
    const buf = new Uint8Array(data.length + headerView.length).fill(0);
    buf.set(headerView, 0);
    buf.set(data, headerView.length);
    console.log(buf);
    if (window.navigator.msSaveOrOpenBlob) {
      window.navigator.msSaveOrOpenBlob(buf.buffer, "blob.pbm");
    } else {
      const a = document.createElement("a");
      document.body.appendChild(a);
      const url = (window.URL ? URL : webkitURL).createObjectURL(
        new Blob([buf]),
        { type: "data:application/octet-stream" }
      );
      a.href = url;
      const date = new Date(Date.now());
      const options = {
        weekday: "short",
        month: "short",
        day: "numeric",
        hour: "numeric",
        minute: "numeric",
        seconds: "numeric",
      };
      a.download =
        "brians-petrol-signs-" +
        date
          .toLocaleString("en-AU", options)
          .replaceAll(",", "-")
          .replace(/\s/g, "") +
        ".pbm";
      a.click();
      setTimeout(() => {
        window.URL.revokeObjectURL(url);
        document.body.removeChild(a);
      }, 0);
    }
  }

  /*
  function post_www_url_encoded(data) {
    const body = new URLSearchParams();
    for (let key in data) {
      body.append(key, data[key]);
    }
    fetch("/command", { method: "POST", body });
  }
  */

  function drawPixel(x, y, val) {
    const data = {
      command: "Y",
      x: x,
      y: y,
      val: val,
    };
    post_www_url_encoded(data);
  }

  function hydrate() {
    console.log(LEDPanelData);

    const panel_x = (current_panel - 1) % 6;
    const panel_y = Math.floor((current_panel - 1) / 6);
    let offset = 16 * panel_x + 16 * panel_width * panel_y;
    let index = offset;
    for (let i = 0; i < 256; i++) {
      document.getElementById(`c${i}`).checked = LEDPanelData[index] === 1;
      index = index + 1;
      if (index % 16 === 0) {
        index = index + panel_width - 16;
      }
    }
  }

  function clearallcheckboxes() {
    document.querySelectorAll("input").forEach((elem) => {
      elem.checked = false;
    });
  }

  function loadCheckboxes() {
    const pbmfile = document.getElementById("input").files[0];
    console.log(pbmfile.name);
    console.log(pbmfile.size);
    const reader = new FileReader();
    reader.addEventListener("loadend", async () => {
      console.log(reader.result);
      const rawbuffer = new Uint8Array(reader.result);
      console.log(rawbuffer);
      let index = 0;

      let startOfCommentsIndex = 3;
      let newlinecounter = 0;
      let startOfDataIndex = 9;

      if (rawbuffer[startOfCommentsIndex] === 0x23) {
        for (let i = startOfCommentsIndex; i < pbmfile.size; i++) {
          if (rawbuffer[i] === 0x0a) newlinecounter = newlinecounter + 1;

          if (newlinecounter === 2) {
            startOfDataIndex = i + 1;
            break;
          }
        }
      }

      for (let i = startOfDataIndex; i < pbmfile.size; i++) {
        const val = rawbuffer[i];
        for (let j = 7; j >= 0; j--) {
          if (val & (1 << j)) {
            LEDPanelData[index] = 1;
          } else {
            LEDPanelData[index] = 0;
          }

          index = index + 1;
        }
      }
      console.log(LEDPanelData);
      document.getElementById("panel" + current_panel).click();
    });
    reader.readAsArrayBuffer(pbmfile);
  }

  function handleLoadFile() {
    handleFiles();
    loadCheckboxes();
  }

  onMount(() => {
    //post_www_url_encoded({ command: "@" }); //clear LEDs
    //const inputElement = document.getElementById("input");
    //inputElement.addEventListener("change", handleLoadFile, false);

    document.getElementById("saveButton").addEventListener("click", (event) => {
      SaveFile();
    });

    document.getElementById("loadButton").addEventListener("click", (event) => {
      document.getElementById("input").click();
      loadCheckboxes();
    });

    document
      .getElementById("clearButton")
      .addEventListener("click", (event) => {
        const data = { command: "@" };
        post_www_url_encoded(data);
        LEDPanelData.fill(0);
        document.getElementById("panel" + current_panel).click();
      });

    document.querySelectorAll("input[type=checkbox]").forEach((checkbox) => {
      checkbox.addEventListener("click", (event) => {
        const ledindex = parseInt(event.target.id.substring(1));
        const panel_x = (current_panel - 1) % 6;
        const panel_y = Math.floor((current_panel - 1) / 6);
        const offset_x = ledindex % 16;
        const offset_y = Math.floor(ledindex / 16);
        const dataIndex =
          offset_x +
          panel_width * offset_y +
          16 * panel_x +
          16 * panel_y * panel_width;
        console.log("ledindex", ledindex, "dataindex", dataIndex);
        if (event.target.checked === true) LEDPanelData[dataIndex] = 1;
        else LEDPanelData[dataIndex] = 0;

        const x = (ledindex % 16) + 16 * panel_x;
        const y = Math.floor(ledindex / 16) + 16 * panel_y;
        console.log("x:", x, " y:", y, "val:", LEDPanelData[dataIndex]);
        console.log(LEDPanelData);
        drawPixel(x, y, LEDPanelData[dataIndex]);
      });
    });

    document.querySelectorAll(".panel").forEach((panel) => {
      panel.addEventListener("click", (event) => {
        document.querySelectorAll(".panel").forEach((p) => {
          p.classList.remove("active");
        });
        event.target.classList.add("active");
        current_panel = parseInt(event.target.innerHTML);
        clearallcheckboxes();
        hydrate();
      });
    });
  });
</script>

<main>
  <section id="menu">
    <ul>
      <li><button id="saveButton">SAVE</button></li>
      <li><button id="loadButton">LOAD</button></li>
      <li class="clearbutton"><button id="clearButton">CLEAR</button></li>
    </ul>
  </section>

  {#each hiddencheckboxclassName as checkboxclassname}
    <input type="checkbox" id={checkboxclassname} name={checkboxclassname} />
  {/each}

  <section id="guts">
    <div id="board" class="grid">
      {#each hiddenlabelclassName as label}
        <label for={label[1]} id={label[0]} class="grid__item" />
      {/each}
    </div>
    <div id="panelselection">
      <h2>PANEL SELECTION</h2>
      <div class="panels">
        {#each panels as panel}
          <button
            class="panel {panel === current_panel ? 'active' : ''}"
            id="panel{panel}">{panel}</button
          >
        {/each}
      </div>
    </div>
  </section>
  <input type="file" id="input" style="display:none;" />
</main>

<style lang="scss">
  #guts {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
  }

  .panels {
    display: grid;
    grid-template-columns: repeat(6, 1fr);
    background-color: #c4c4c4;
  }

  .panel:hover {
    background-color: #e4e4e4;
  }

  .panel {
    padding: 0;
    margin: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 80px;
    background-color: #fff;
    border: 2px solid #c4c4c4;
    font-size: 2rem;
    color: #333;
    text-shadow: 0 1px rgba(255, 255, 255, 0.4);
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.05);
    background-image: linear-gradient(
      to bottom,
      transparent,
      transparent,
      50%,
      rgba(0, 0, 0, 0.04)
    );
  }

  .active {
    border: 2px solid green;
    color: red;
    background-color: lightcyan;
  }

  ul button {
    margin: 0;
  }

  ul {
    padding: 0;
    margin-bottom: 5px;
  }

  .grid {
    display: -ms-grid;
    -ms-grid-columns: repeat(16, 1fr);
    -ms-grid-rows: auto;
    display: grid;
    grid-template-columns: repeat(16, 1fr);
    grid-template-rows: auto;
  }

  /* 
.buttons {
  max-width: 800px;
  margin: 30px auto;
  box-sizing: border-box;

  button {
    width: 6rem;
  }
}
*/

  .grid__item::before {
    content: "";
    display: block;
    padding-top: 100%;
  }

  /*
  body {
    padding: 0;
    margin: 0;
    background-color: #212121;
  }
  */

  .grid {
    max-width: 600px;
    /*margin: 30px auto;*/
    box-sizing: border-box;
    border: 10px solid #2a351f;
  }
  .grid__item {
    /*background-color: #eeedd3;*/
    background: #c0c0c0;
    border: 2px outset #ececec;
    transition: all 400ms linear;
  }

  $columns: 255;

  @for $i from 0 through $columns {
    #c#{$i}:checked ~ section .grid #s#{$i} {
      background-color: green;
      margin: 0px;
      transform: scale(1);
    }
  }

  /*

  #c1:checked ~ .grid #s1 {
    background-color: green;    
    margin: 0px;  
    transform: scale(1.0);
  
  }
  
  #c2:checked ~ .grid #s2 {
    background-color: green;
    margin: 0px;
    transform: scale(1.0);
  }
  */

  input {
    visibility: hidden;
    position: absolute;
    top: -99px;
    left: -99px;
  }

  li.clearbutton > :global(:last-child) {
    margin-left: 350px;
  }

  li {
    display: inline-block;
  }

  /*
  .selected {
    color: red;
  }
  */
</style>
