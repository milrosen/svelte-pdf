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

  /* 
  ##Device = Tablets, Ipads (portrait)
  ##Screen = B/w 768px to 1024px
  */

  @media (min-width: 768px) and (max-width: 1024px) {
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
    canvas {
      width: 100%;
      height: 100%;
    }
  }
</style>
