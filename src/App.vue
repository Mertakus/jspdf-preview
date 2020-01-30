<template>
  <div>
    <div class="top-bar">
      <button
        class="btn"
        id="prev-page"
        @click="prevPage"
      >
        Prev Page
      </button>
      <button
        class="btn"
        id="next-page"
        @click="nextPage"
      >
        Next Page
      </button>
      <span class="page-info">
        Page <span id="page-num"></span> of <span id="page-count"></span>
      </span>
    </div>

    <canvas id="pdf-render"></canvas>

  </div>
</template>

<script>

  import PDFJS from 'pdfjs-dist'
  import url from '../src/assets/pdf/example.pdf'

  export default {
    data() {
      return {
        pdfDoc: null,
        pageNum: 1,
        pageIsRendering: false,
        pageNumIsPending: null
      }
    },
    methods: {
      renderPage(num) {
        this.pageIsRendering = true;

        const canvas = document.querySelector('#pdf-render')
        const ctx = canvas.getContext('2d');

        // Get page
        this.pdfDoc.getPage(num).then(page => {

          // Set scale
          const viewport = page.getViewport({ scale: 1 });
          canvas.height = viewport.height;
          canvas.width = viewport.width;

          const renderCtx = {
            canvasContext: ctx,
            viewport: viewport
          };

          page.render(renderCtx).promise
            .then(() => {
              this.pageIsRendering = false;
              // page.cleanup()

              if (this.pageNumIsPending !== null) {
                this.renderPage(this.pageNumIsPending);
                this.pageNumIsPending = null;
              }
            })
            .catch(err => {
              console.log('ERR: ', err);
              this.showPDF(url)
            })

          // Output current page
          document.querySelector('#page-num').textContent = num;
        });
      },
      showPDF(pdf_url) {
        PDFJS
          .getDocument({ url: pdf_url })
          .promise.then(pdfDoc_ => {
            this.pdfDoc = pdfDoc_;
            // console.log(this.pdfDoc);

            document.querySelector('#page-count').textContent = this.pdfDoc.numPages;

            this.renderPage(this.pageNum);
          })
          .catch(err => {
            // Display error
            console.log('ERRROR: ', err);
            const div = document.createElement('div');
            div.className = 'error';
            div.appendChild(document.createTextNode(err.message));
            document.querySelector('body').insertBefore(div, canvas);
            // Remove top bar
            document.querySelector('.top-bar').style.display = 'none';
          });
      },
      // Check for pages rendering
      queueRenderPage(num) {
        if (this.pageIsRendering) {
          this.pageNumIsPending = num;
        } else {
          this.renderPage(num);
        }
      },
      prevPage() {
        if (this.pageNum <= 1) {
          return;
        }
        this.pageNum--;
        this.queueRenderPage(this.pageNum);
      },
      nextPage() {
        if (this.pageNum >= this.pdfDoc.numPages) {
          return;
        }
        this.pageNum++;
        this.queueRenderPage(this.pageNum);
      }
    },
    created() {
      // Get Document
      this.showPDF(url)
    },
  }

</script>

<style>
  * {
    margin: 0;
    padding: 0;
    font-family:'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }

  #pdf-render {
    border: 1px solid gray;
  }

  .top-bar {
    background: #333;
    color: #fff;
    padding: 1rem;
  }

  .btn {
    background: #419994;
    border-radius: 4px;
    color: #fff;
    border: none;
    outline: none;
    cursor: pointer;
    padding: 0.7rem 2rem;
    margin: 2px;
  }

  .btn:hover {
    opacity: 0.9;
  }

  .page-info {
    margin-left: 1rem;
  }

  .error {
    background: orangered;
    color: #fff;
    padding: 1rem;
  }
</style>


<!-- <template>
  <div id="app">
    <img
      alt="Vue logo"
      src="./assets/logo.png"
    >
    <button @click="downloadPDF">downloadPDF</button>

    <input
      type="text"
      v-model="personData.basic.firstname"
    >
    <input
      type="text"
      v-model="personData.basic.surname"
    >
    <input
      type="text"
      v-model="personData.basic.position"
    >

    <p>firstname: {{ personData.basic.firstname }}, surname: {{ personData.basic.surname }}, position:
      {{ personData.basic.position }}</p>
    <hr>

    <div id="pdf-main-container">
      <div id="pdf-loader">Loading document ...</div>
      <div id="pdf-contents">
        <div id="pdf-meta">
          <div id="pdf-buttons">
            <button
              @click="prevPage"
              id="pdf-prev"
            >Previous</button>
            <button
              @click="nextPage"
              id="pdf-next"
            >Next</button>
          </div>
          <div id="page-count-container">Page <div id="pdf-current-page"></div> of <div id="pdf-total-pages">
            </div>
          </div>
        </div>
        <canvas
          id="pdf-canvas"
          width="500"
        ></canvas>
        <div id="page-loader">Loading page ...</div>
        <a
          id="download-image"
          @click="download"
        >Download PNG</a>
      </div>
    </div>

  </div>
</template>

<script>
  import jsPDF from 'jspdf'
  import debounce from 'lodash.debounce';

  export default {
    name: 'app',
    data() {
      return {
        personData: {
          basic: {
            firstname: 'Reinier',
            surname: 'van der Galien',
            position: ''
          }
        },
        typingTimer: null,
        doneTypingInterval: 2500,

        file: null,
        pdf_doc: null,
        current_page: 0,
        total_pages: 0,
        page_rendering_in_progress: 0,

        downloadPressed: false
      }
    },
    components: {

    },
    methods: {
      log() {
        console.log('__CANVAS');
      },
      download() {
        this.downloadPressed = true;
        console.log('downloadpressed: ', this.downloadPressed);
        // this.downloadPressed = false;
      },
      showPDF(pdf_url) {
        $("#pdf-loader").show();

        PDFJS.getDocument({ url: pdf_url }).then((pdf_doc) => {
          this.pdf_doc = pdf_doc;
          this.total_pages = this.pdf_doc.numPages;

          // Hide the pdf loader and show pdf container in HTML
          $("#pdf-loader").hide();
          $("#pdf-contents").show();
          $("#pdf-total-pages").text(this.total_pages);

          // Show the first page
          this.showPage(1);
        }).catch(function (error) {
          // If error re-show the upload button
          $("#pdf-loader").hide();
          $("#upload-button").show();

          alert(error.message);
        });;
      },
      showPage(page_no) {
        this.page_rendering_in_progress = 1;
        this.current_page = page_no;

        let __CANVAS = $('#pdf-canvas').get(0);
        let __CANVAS_CTX = __CANVAS.getContext('2d');

        // Disable Prev & Next buttons while page is being loaded
        $("#pdf-next, #pdf-prev").attr('disabled', 'disabled');

        // While page is being rendered hide the canvas and show a loading message
        $("#pdf-canvas").hide();
        $("#page-loader").show();
        $("#download-image").hide();

        // Update current page in HTML
        $("#pdf-current-page").text(page_no);

        // Fetch the page
        this.pdf_doc.getPage(page_no).then((page) => {

          if (window.devicePixelRatio > 1) {
            console.log('retina');

            // As the canvas is of a fixed width we need to set the scale of the viewport accordingly
            // var scale_required = __CANVAS.width / page.getViewport(2).width;
            // var viewport = page.getViewport(1); // 2 is the 'scale' attr

            // // Get viewport of the page at required scale
            // var viewport = page.getViewport(scale_required);

            var viewport = page.getViewport(1.35);

            __CANVAS.width = 800;
            __CANVAS.style.width = "400px";

            // Set canvas height
            __CANVAS.height = viewport.height;

            var renderContext = {
              canvasContext: __CANVAS_CTX,
              viewport: viewport
            };
          } else {
            // As the canvas is of a fixed width we need to set the scale of the viewport accordingly
            var scale_required = __CANVAS.width / page.getViewport(1).width;

            // Get viewport of the page at required scale
            var viewport = page.getViewport(scale_required);

            // Set canvas height
            __CANVAS.height = viewport.height;

            var renderContext = {
              canvasContext: __CANVAS_CTX,
              viewport: viewport
            };
          }

          // Render the page contents in the canvas
          page.render(renderContext).then(() => {
            this.page_rendering_in_progress = 0;

            // Re-enable Prev & Next buttons
            $("#pdf-next, #pdf-prev").removeAttr('disabled');

            // Show the canvas and hide the page loader
            $("#pdf-canvas").show();
            $("#page-loader").hide();
            $("#download-image").show();
          });
        });
      },
      prevPage() {
        // Previous page of the PDF
        if (this.current_page != 1)
          this.showPage(--this.current_page);
      },
      nextPage() {
        // Next page of the PDF
        if (this.current_page != this.total_pages)
          this.showPage(++this.current_page);
      },
      downloadPDF() {
        var doc = new jsPDF()

        doc.text(this.personData.basic.firstname, 10, 10)
        doc.text(this.personData.basic.surname, 10, 20)
        doc.text(this.personData.basic.position, 10, 30)

        this.file = new File([doc.output('blob')], "preview", { type: 'application/pdf' });
        this.showPDF(URL.createObjectURL(this.file));
        // console.log(this.file)

        if (this.downloadPressed) {
          doc.save('a4.pdf')
        }

      },
    },
    created() {
      var doc = new jsPDF()
      doc.text(this.personData.basic.firstname, 10, 10)
      doc.text(this.personData.basic.surname, 10, 20)
      doc.text(this.personData.basic.position, 10, 30)
      this.file = new File([doc.output('blob')], "preview", { type: 'application/pdf' });
      this.showPDF(URL.createObjectURL(this.file));

      document.addEventListener('keyup', (e) => {
        clearTimeout(this.typingTimer);
        if (e.code) {
          this.typingTimer = setTimeout(this.downloadPDF, this.doneTypingInterval);
        }
      });
    },
  }
</script>

<style>
  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
  }

  #upload-button {
    width: 150px;
    display: block;
    margin: 20px auto;
  }

  #file-to-upload {
    display: none;
  }

  #pdf-main-container {
    width: 500px;
    margin: 20px auto;
  }

  #pdf-loader {
    display: none;
    text-align: center;
    color: #999999;
    font-size: 13px;
    line-height: 100px;
    height: 100px;
  }

  #pdf-contents {
    display: none;
  }

  #pdf-meta {
    overflow: hidden;
    margin: 0 0 20px 0;
  }

  #pdf-buttons {
    float: left;
  }

  #page-count-container {
    float: right;
  }

  #pdf-current-page {
    display: inline;
  }

  #pdf-total-pages {
    display: inline;
  }

  #pdf-canvas {
    border: 1px solid rgba(0, 0, 0, 0.2);
    box-sizing: border-box;
  }

  #page-loader {
    height: 100px;
    line-height: 100px;
    text-align: center;
    display: none;
    color: #999999;
    font-size: 13px;
  }

  #download-image {
    width: 150px;
    display: block;
    margin: 20px auto 0 auto;
    font-size: 13px;
    text-align: center;
  }
</style> -->