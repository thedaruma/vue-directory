<template lang="html">
  <div class="overlay" @click="closeModal">
<div class="panel" @click.stop>    <div class="panel-body"><form class="form-horizontal" v-if="loggedIn">
      <fieldset>
          <div class="form-group">
              <label for="inputEmail" class="col-lg-2 control-label">First Name</label>
              <div class="col-lg-7">
                  <input type="text" class="form-control" v-model="employee.firstName" id="firstName" placeholder="First Name">
              </div>
          </div>
          <div class="form-group">
              <label for="inputEmail" class="col-lg-2 control-label">Last Name</label>
              <div class="col-lg-7">
                  <input type="text" class="form-control" v-model="employee.lastName" id="lastName" placeholder="lastName">
              </div>
          </div>
          <!--This is where the image happens  -->
          <div class="form-group" v-if="!reset">
              <label for="image" class="col-lg-2 control-label">Image Link</label>
              <div class="col-lg-10">
                  <div class="" v-if="image">
                      <img :src="image" id="target" v-if="!reset"/>
                      <button type="button" name="button" @click="removeImage">Remove Image</button>
                  </div>
                  <div v-show="!image">
                      <form action="post" id="image-form"  enctype="multipart/form-data">
                          <input id="image-input" style="display:block" :ref="" type="file" @change="onFileChange" name="file">
                      </form>
                  </div>
              </div>
              <div class="col-lg-10" v-show="false">
                <!-- display the cropped image -->
                <canvas id="canvas"></canvas>
                <input id="png" type="hidden" />
              </div>
              <!-- (x:{{crop.x}},y:{{crop.y}},x2:{{crop.x2}},y2:{{crop.y2}},h:{{crop.h}},w:{{crop.w}}) -->
          </div>
          <!--  End image happenings -->

          <div class="form-group">
              <div class="col-lg-10 col-lg-offset-2">
                  <button class="btn btn-default" @click="closeModal">Cancel</button>
                  <button class="btn btn-primary" @click="submit">Submit</button>
              </div>
          </div>
      </fieldset>
  </form></div></div>
</div>
</template>

<script>
import {
    EmployeeHttp
} from '../../mixins/employee';
import {
    eventBus
} from '../../main.js';
export default {
    mixins:[EmployeeHttp],
    computed: {
        loggedIn() {
            return this.$store.getters.loggedIn;
        }
    },
    data() {
        return {
            crop_max_width: 300,
            crop_max_height: 300,
            imageObj: '',
            employee: {
                firstName: '',
                lastName: '',
                img: ''
            },
            crop: {
                x: '',
                y: '',
                x2: '',
                y2: '',
                h: '',
                w: ''
            },
            image: '',
            reset: false
        }
    },
    methods: {
      closeModal(e){
        e.preventDefault();
        this.$emit('close',true);
      },
        onFileChange(e) {
            //takes the base 64 string of the file
            var files = e.target.files || e.dataTransfer.files;
            if (!files.length)
                return;
            console.log(files[0]);
            this.createImage(files[0]);
        },
        createImage(file) {
            var image = new Image();
            var reader = new FileReader();
            reader.readAsDataURL(file);
            reader.onload = (e) => {
                this.image = e.target.result;
                setTimeout(() => {
                    $("#target, #preview").html("").append("<img src=\"" + e.target.result + "\" alt=\"\" />");
                    $('#target').Jcrop({
                        trackDocument: true,
                        aspectRatio: 1,
                        setSelect: getDimensions(),
                        onSelect: this.updateCoords,
                        onChange: this.updateCoords,
                        boxWidth: this.crop_max_width,
                        boxHeight: this.crop_max_height
                    });
                }, .500)
            };

            function getDimensions() {
                return [
                    0,
                    0,
                    Math.round($('#target').width() / 2),
                    Math.round($('#target').height() / 2)
                ]
            }
        },
        updateCoords(c) {
            this.crop.x = c.x,
                this.crop.y = c.y,
                this.crop.y2 = c.y2,
                this.crop.x2 = c.x2,
                this.crop.w = c.w,
                this.crop.h = c.h
        },
        drawCanvas(c) {
            //canvas is never seen
            this.imageObj = $("#target")[0];
            var canvas = $("#canvas")[0];
            canvas.width = this.crop.w;
            canvas.height = this.crop.h;

            var context = canvas.getContext("2d");
            context.drawImage(this.imageObj, this.crop.x, this.crop.y, this.crop.w, this.crop.h, 0, 0, canvas.width, canvas.height);
        },
        removeImage: function(e) {
            this.resetImageField();
        },
        resetImageField() {
            this.reset = true;
            setTimeout(() => {
                this.reset = false;
                this.image = '';
                // console.log(this.reset);
                //Wipes the images input field clean.  Without this, the filename still remains.
            }, 500)
        },
        submitImage(employee) {
            return new Promise((resolve, reject) => {

                let formData = new FormData($('image-form')[0]);
                formData.append('token', localStorage.getItem('token'));
                formData.append('file', $("#canvas")[0].toDataURL('image/jpeg'));
                formData.append('uniqueId', employee.uniqueId);
                formData.append('about', employee.about);
                formData.append('firstName', employee.firstName);
                formData.append('lastName', employee.lastName);
                formData.append('title', employee.title);
                formData.append('branchId', employee.branchId);
                formData.append('phone', employee.phone);
                formData.append('email', employee.email);
                formData.append('id', employee.id);
                this.$http.post('http://localhost/api/routes/images/upload.php', formData, {
                        emulateJSON: true,
                        upload: {
                            onprogress: (e) => {
                                console.log(e);
                                if (e.lengthComputable) {

                                    this.progress[this.count].loaded = e.loaded;
                                    this.progress[this.count].total = e.total;
                                }
                            }
                        }
                    })
                    .then(response => {
                        if (response.ok) {
                          console.log(response);
                            let newEmployee = JSON.parse(response.bodyText);
                            eventBus.$emit('employeeAdded', newEmployee);
                            setTimeout(() => {
                                eventBus.$emit('uploading', false);
                            }, 2000)
                            resolve()
                        } else {
                            reject(response.statusText);
                        }
                    });
            })

        },
        submit(e) {
            e.preventDefault();
            eventBus.$emit('uploading', true);
            this.$http.post('http://localhost/api/routes/employees/get-post.php', this.employee, {
                    emulateJSON: true
                })
                .then(response => {
                    if (response.ok) {
                        let submittedEmployee = response.body; //returns the employee just posted
                        console.log(submittedEmployee);
                        //draw the canvas to crop the image on the client-side.
                        this.drawCanvas();
                        setTimeout(() => {
                            this.submitImage(submittedEmployee).then(data => {
                                this.resetImageField();
                            });

                            this.employee.firstName = '';
                            this.employee.lastName = '';
                            this.employee.img = '';
                        }, 500)


                    } else {
                        console.error('there was an error');
                    }

                })
        },
    }
}
</script>

<style scoped>
#target {
    /*min-height:300px;
  max-height:300px;*/
}
.panel{
  width:50%;
  min-height:200px;
  border-radius:7px;
  -webkit-box-shadow: 0 1px 4px rgba(0, 0, 0, 0.3);
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.3);
}
.overlay{
  background-color:rgba(0,0,0,.5);
  display:flex;
  position:absolute;
  z-index:2000;
  height:100%;
  width:100%;
  top:0;
  left:0;
  justify-content:center;
  align-items:center;
}

.form-horizontal {
    min-height: 230px;
}
</style>
