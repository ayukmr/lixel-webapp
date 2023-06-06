<script>
  import axios  from 'axios';
  import PNGLib from 'pnglib';

  const root   = 'http://localhost:5000'
  const params = new URLSearchParams(location.search);

  let updatePixels = [];
  let currentColor = '#ff0000';

  let newCanvasW = 16;
  let newCanvasH = 16;

  let canvas;
  let canvasId = params.get('id');

  $: {
    // get canvas on id update
    getCanvas();
    canvasId = canvasId;
  }

  // get canvas every 0.5 secs
  setInterval(getCanvas, 500);

  // update canvas every 0.025 secs
  setInterval(updateCanvas, 25);

  // get canvas content
  function getCanvas() {
    if (canvasId) {
      axios.get(`${root}/canvas/${canvasId}/content`)
        .then((res) => {
          canvas = res.data.content;
          history.replaceState({}, '', `?id=${canvasId}`);
        });
    }
  }

  // create new canvas
  function createCanvas() {
    let data = {
      content: createArray(newCanvasW, newCanvasH, '#fff')
    };

    axios.post(`${root}/canvas`, data)
      .then((res) => {
        canvasId = res.data.id;
      });
  }

  // update pixel on canvas
  function updatePixel(x, y, color) {
    if (canvas[y][x] != color) {
      updatePixels.push({ x, y, color });
    }
  }

  // update canvas
  function updateCanvas() {
    if (updatePixels.length) {
      axios.put(`${root}/canvas/${canvasId}/content`, { pixels: updatePixels })
        .then(() => {
          updatePixels = [];
          getCanvas();
        });
    }
  }

  // delete canvas
  function deleteCanvas() {
    if (canvasId) {
      axios.delete(`${root}/canvas/${canvasId}`)
        .then(() => {
          canvas   = null;
          canvasId = null;

          history.replaceState({}, '', '/');
        });
    }
  }

  // export canvas as png
  function exportCanvas() {
    if (canvas) {
      // create png
      var png = new PNGLib(canvas[0].length, canvas.length, 256);

      canvas.forEach((row, y) => {
        row.forEach((color, x) => {
          let { r, g, b } = colorToRGB(color);

          // add color to buffer
          png.buffer[png.index(x, y)] = png.color(r, g, b);
        });
      });

      let download = document.createElement('a');

      // create download element
      download.href = `data:image/png;base64,${png.getBase64()}`;
      download.download = `lixel-${canvasId}.png`;

      download.click();
    }
  }

  // create 2d array
  function createArray(sizeW, sizeH, value) {
    var arr = [];

    for(let y = 0; y < sizeH; y++) {
      arr[y] = [];

      for(let x = 0; x < sizeW; x++) {
        arr[y][x] = value;
      }
    }

    return arr;
  }

  // convert css colors to rgb
  function colorToRGB(color) {
    let elem = document.createElement('div');

    elem.style.color = color;
    document.body.appendChild(elem);

    let rgbString = window.getComputedStyle(elem).color;
    document.body.removeChild(elem);

    let result = /(\d{1,3}), *(\d{1,3}), *(\d{1,3})/i.exec(rgbString);

    return {
      r: parseInt(result[1]),
      g: parseInt(result[2]),
      b: parseInt(result[3]),
    };
  }
</script>

<div>
  <!-- color input -->
  <input
    bind:value={currentColor}

    style:position="fixed"
    style:right="5px"
    style:top="5px"

    style:width="130px"
  />

  <!-- color circle -->
  <div
    style:position="fixed"
    style:right="8.5px"
    style:top="8.5px"

    style:background={currentColor}
    style:width="13px"
    style:height="13px"
  />

  <!-- canvas id input -->
  <input
    bind:value={canvasId}

    style:position="fixed"
    style:left="5px"
    style:top="5px"
  />

  <!-- new canvas width -->
  <input
    bind:value={newCanvasW}

    style:position="fixed"
    style:left="5px"
    style:top="28px"

    style:width="20px"
    style:text-align="center"
  />

  <!-- new canvas height -->
  <input
    bind:value={newCanvasH}

    style:position="fixed"
    style:left="32px"
    style:top="28px"

    style:width="20px"
    style:text-align="center"
  />

  <!-- create new canvas -->
  <button
    style:position="fixed"
    style:left="59px"
    style:top="28px"

    on:click={createCanvas}
  >
    create new
  </button>

  <!-- export canvas -->
  <button
    style:position="fixed"
    style:left="5px"
    style:bottom="5px"

    on:click={exportCanvas}
  >
    export canvas
  </button>

  <!-- delete canvas -->
  <button
    style:position="fixed"
    style:right="5px"
    style:bottom="5px"

    on:click={deleteCanvas}
  >
    delete canvas
  </button>
</div>

<div
  style:cursor="crosshair"
  style:user-select="none"
>
  {#if canvas}
    <!-- draw canvas -->
    {#each canvas as row, y}
      <div style:display="flex">
        {#each row as color, x}
          <!-- draw pixel -->
          <div
            style:display="inline-block"
            style:background={color}

            style:width="35px"
            style:height="35px"

            on:mousedown={() => {
              updatePixel(x, y, currentColor);
            }}

            on:mouseenter={(e) => {
              if (e.buttons === 1) {
                updatePixel(x, y, currentColor);
              }
            }}

            on:contextmenu={(e) => {
              e.preventDefault();
              updatePixel(x, y, '#fff');
            }}
          />
        {/each}
      </div>
    {/each}
  {:else}
    No canvas!
  {/if}
</div>
