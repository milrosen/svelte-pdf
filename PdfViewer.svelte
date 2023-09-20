<script>
  import { onDestroy, tick } from "svelte";
  import * as pdfjs from "pdfjs-dist";
  import { onPrint, calcRT, getPageText, savePDF } from "./utils/Helper.svelte";
  import Tooltip from "./utils/Tooltip.svelte";

  export let url;
  export let data;
  export let scale = 1.8;
  export let pageNum = 1; //must be number
  export let showButtons = [
    "navigation",
    "zoom",
    "print",
    "rotate",
    "download",
    "autoflip",
    "timeInfo",
    "pageInfo",
  ]; //array
  export let showBorder = true; //boolean
  export let totalPage = 0;
  export let downloadFileName = "";

  pdfjs.GlobalWorkerOptions.workerSrc = new URL(
    "pdfjs-dist/build/pdf.worker.js",
    import.meta.url
  );

  let canvas;
  let pageCount = 0;
  let pdfDoc = null;
  let pageRendering = false;
  let pageNumPending = null;
  let rotation = 0;
  let pdfContent = "";
  let readingTime = 0;
  let interval;
  let secondInterval;
  let pages = [];
  let password = "";
  let passwordError = false;
  let passwordMessage = "";
  let isInitialised = false;

  const renderPage = (num) => {
    pageRendering = true;
    // Using promise to fetch the page
    pdfDoc.getPage(num).then(function (page) {
      let viewport = page.getViewport({ scale: scale, rotation: rotation });
      const canvasContext = canvas.getContext("2d");
      canvas.height = viewport.height;
      canvas.width = viewport.width;

      // Render PDF page into canvas context
      let renderContext = {
        canvasContext,
        viewport,
      };
      let renderTask = page.render(renderContext);

      // Wait for rendering to finish
      renderTask.promise.then(function () {
        pageRendering = false;
        if (pageNumPending !== null) {
          // New page rendering is pending
          // renderPage(pageNumPending);
          if (pageNum < pdfDoc.totalPage) {
            pages[pageNum] = canvas;
            pageNum++;
            pdfDoc.getPage(pageNum).then(renderPage);
          } else {
            for (let i = 1; i < pages.length; i++) {
              canvas.appendChild(pages[i]);
            }
          }
          pageNumPending = null;
        }
      });
    });
  };

  const queueRenderPage = (num) => {
    if (pageRendering) {
      pageNumPending = num;
    } else {
      renderPage(num);
    }
  };

  /**
   * Displays previous page.
   */
  export const prevPage = () => {
    if (pageNum <= 1) {
      return;
    }
    pageNum--;
    queueRenderPage(pageNum);
  };

  /**
   * Displays next page.
   */
  export const nextPage = () => {
    if (pageNum >= pdfDoc.numPages) {
      return;
    }
    pageNum++;
    queueRenderPage(pageNum);
  };

  /**
   * Asynchronously downloads PDF.
   */

  const initialLoad = async () => {
    let loadingTask = pdfjs.getDocument({
      ...(url && { url }),
      ...(data && { data }),
      ...(password && { password }),
    });
    loadingTask.promise
      .then(async function (pdfDoc_) {
        pdfDoc = pdfDoc_;
        passwordError = false;
        await tick();

        totalPage = pdfDoc.numPages;
        if (showButtons.length) {
          for (let number = 1; number <= totalPage; number++) {
            // Extract the text
            getPageText(number, pdfDoc).then(function (textPage) {
              // Show the text of the page in the console
              pdfContent = pdfContent.concat(textPage);
              readingTime = calcRT(pdfContent);
            });
          }
        }
        isInitialised = true;
      })
      .catch(function (error) {
        passwordError = true;
        passwordMessage = error.message;
      });
  };
  initialLoad();
  $: {
    if (isInitialised) queueRenderPage(pageNum);
  }

  //Download pdf function
  const downloadPdf = ({ url: fileURL, data }) => {
    let fileName =
      downloadFileName ||
      (fileURL && fileURL.substring(fileURL.lastIndexOf("/") + 1));
    savePDF({ fileURL, data, name: fileName });
  };
  //prevent memory leak
  onDestroy(() => {
    clearInterval(interval);
    clearInterval(secondInterval);
  });

  let pageWidth;
  let pageHeight;
</script>

<svelte:window bind:innerWidth={pageWidth} bind:innerHeight={pageHeight} />

<canvas bind:this={canvas} width={pageWidth} height={pageHeight} />

<style>
  :global(html) {
    scroll-behavior: smooth;
  }
  .parent {
    display: flex;
    flex-direction: column;
    margin: 0 1.25rem;
  }
  .control-start {
    padding: 1.25rem;
  }
  .line {
    display: flex;
    flex-direction: row;
    font-family: Georgia, Cambria, "Times New Roman", Times, serif;
    border-top-width: 0px;
    border-right-width: 0px;
    border-bottom-width: 1px;
    border-left-width: 0px;
    border-color: #4fd1c5;
    border-style: dotted;
    margin-bottom: 0.75rem;
    padding-top: 0.5rem;
    padding-bottom: 0.5rem;
    justify-content: center;
  }
  .button-control {
    display: flex;
    flex-direction: row;
    padding: 0.5rem;
    margin: 0.75rem;
    border-radius: 0.25rem;
    overflow: hidden;
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1),
      0 4px 6px -2px rgba(0, 0, 0, 0.05);
    border-left-width: 1px;
    border-bottom-width: 1px;
    border-right-width: 1px;
    cursor: pointer;
  }
  .viewer {
    border-width: 1px;
    border-color: #000;
    border-style: solid;
  }
  .icon {
    height: 1.25rem;
    width: 1.25rem;
    fill: currentColor;
    color: #38b2ac;
  }
  .disabled {
    cursor: not-allowed;
    box-shadow: none;
  }
  #topBtn {
    position: fixed;
    bottom: 10px;
    float: right;
    right: 10%;
    left: 90%;
    max-width: 30px;
    width: 100%;
    border-color: #000;
    background-color: #fff;
    padding: 0.5px;
    border-radius: 9999px;
  }
  #topBtn:hover {
    background-color: #000;
    color: #fff;
  }
  /* 
  ##Device = Tablets, Ipads (portrait)
  ##Screen = B/w 768px to 1024px
  */

  @media (min-width: 768px) and (max-width: 1024px) {
    .parent {
      margin: 0;
    }
    .control-start {
      padding: 0;
    }
    .line {
      justify-content: center;
    }
    .button-control {
      display: flex;
      flex-direction: row;
      padding: 0.5rem;
      margin: 0.5rem;
      border-radius: 0.25rem;
      overflow: hidden;
      box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1),
        0 4px 6px -2px rgba(0, 0, 0, 0.05);
      border-left-width: 1px;
      border-bottom-width: 1px;
      border-right-width: 1px;
      cursor: pointer;
    }
    canvas {
      width: 100%;
      height: 100%;
    }
  }
  /* 
  ##Device = Low Resolution Tablets, Mobiles (Landscape)
  ##Screen = B/w 481px to 767px
  */

  @media (min-width: 481px) and (max-width: 767px) {
    .parent {
      margin: 0;
    }
    .control-start {
      padding: 0;
    }
    .line {
      justify-content: center;
    }
    .button-control {
      display: flex;
      flex-direction: row;
      padding: 0.5rem;
      margin: 0.5rem;
      border-radius: 0.25rem;
      overflow: hidden;
      box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1),
        0 4px 6px -2px rgba(0, 0, 0, 0.05);
      border-left-width: 1px;
      border-bottom-width: 1px;
      border-right-width: 1px;
      cursor: pointer;
    }
    canvas {
      width: 100%;
      height: 100%;
    }
  }

  /* 
  ##Device = Most of the Smartphones Mobiles (Portrait)
  ##Screen = B/w 320px to 479px
  */

  @media (min-width: 320px) and (max-width: 480px) {
    .parent {
      margin: 0;
    }
    .control-start {
      padding: 0;
    }
    .line {
      justify-content: center;
    }
    .button-control {
      display: flex;
      flex-direction: row;
      padding: 0.4rem;
      margin: 0.4rem;
      border-radius: 0.25rem;
      overflow: hidden;
      box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1),
        0 4px 6px -2px rgba(0, 0, 0, 0.05);
      border-left-width: 1px;
      border-bottom-width: 1px;
      border-right-width: 1px;
      cursor: pointer;
    }
    canvas {
      width: 100%;
      height: 100%;
    }
  }
</style>
