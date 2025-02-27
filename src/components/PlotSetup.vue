<template>
    <div>
        <li class="type">
            <div v-b-toggle.plotsetupcontent>
                <a class="section"> Plots Setup
                    <i class="expand fas fa-caret-down"></i></a>
            </div>
        </li>
        <b-collapse class="menu-content collapse out" id="plotsetupcontent" visible>
            <ul class="colorpicker plot-wrapper">
                <template v-for="(field, index) in state.expressions">
                    <li :key="'field'+index" class="field plotsetup">
                        <input class="plotname" v-model.lazy="field.name" v-debounce="1000">
                        <select v-model.number="field.axis">
                            <option v-bind:key="'axisnumber'+axis" v-for="axis in state.allAxis">{{axis}}</option>
                        </select>
                        <select :style="{color: field.color}" v-model="field.color">
                            <option
                                v-bind:key="'axisColor'+color"
                                :style="{color: color}"
                                v-bind:value="color"
                                v-for="color in state.allColors">■
                            </option>
                        </select>
                        <a class="remove-button" @click="$eventHub.$emit('togglePlot', field.name)">
                            <i class="expand fas fa-trash" title="Remove data"></i>
                        </a>
                    </li>
                    <li :key="'field'+index+'err'" v-if="state.expressionErrors[index]" class="error">
                        <i class="fas fa-exclamation-circle error"
                           :title="state.expressionErrors[index]">
                        </i>
                        {{state.expressionErrors[index]}}
                    </li>
                </template>
                <li v-if="state.expressions.length === 0">  Please plot something first.</li>
            </ul>
            <!-- BUTTONS -->
            <div class="btns-wrapper">
            <button class="add-expression" @click="createNewExpression()">
            <i class="fa fa-plus" aria-hidden="true"></i>Add Expression</button>
            <button class="save-preset" v-if="state.expressions.length > 0" v-b-modal.modal-prevent-closing>
            <i class="fa fa-check-circle" aria-hidden="true"></i>Save Preset</button>
                <button
                    class="save-preset"
                    v-if="state.expressions.length > 0"
                    v-b-modal.modal-prevent-closing
                    @click="$eventHub.$emit('clearPlot')"
                >
                    <i class="fa fa-ban" aria-hidden="true"></i>
                    clear
                </button>
            </div>
        </b-collapse>
        <!-- MODAL -->
        <b-modal
            id="modal-prevent-closing"
            ref="modal"
            @show="resetModal"
            @hidden="resetModal"
            @ok="handleOk"
        >
            <form ref="form" @submit.stop.prevent="handleOk">
                <b-form-group
                    label="New Preset Name"
                    label-for="name-input"
                >
                    <b-form-input
                        id="name-input"
                        v-model="name"
                        placeholder="Attitude/OtherRoll"
                        required
                    ></b-form-input>
                </b-form-group>
            </form>
        </b-modal>
    </div>
</template>
<script>
import {store} from './Globals.js'
import debounce from 'v-debounce'

export default {
    name: 'plotSetup',
    data () {
        return {
            state: store,
            name: ''
        }
    },
    methods: {
        createNewExpression () {
            this.state.plotOn = true
            this.$nextTick(() => {
                this.state.expressions.push({
                    name: '1+1',
                    color: this.getFirstFreeColor(),
                    axis: this.getFirstFreeAxis()
                })
            })
        },
        // TODO: this is duplicated in Plotly.vue, refactor it out!
        getFirstFreeAxis () {
            // get free axis number
            for (let i of this.state.allAxis) {
                let taken = false
                for (let field of this.state.expressions) {
                    // eslint-disable-next-line
                    if (field.axis == i) {
                        taken = true
                    }
                }
                if (!taken) {
                    return i
                }
            }
            return this.state.allAxis.length - 1
        },
        getFirstFreeColor () {
            // get free color
            for (let i of this.state.allColors) {
                let taken = false
                for (let field of this.state.expressions) {
                    // eslint-disable-next-line
                    if (field.color == i) {
                        taken = true
                    }
                }
                if (!taken) {
                    return i
                }
            }
            return this.state.allColors[this.state.allColors.length - 1]
        },
        savePreset (name) {
            let myStorage = window.localStorage
            let saved = myStorage.getItem('savedFields')
            if (saved === null) {
                saved = {}
            } else {
                saved = JSON.parse(saved)
            }
            saved[name] = []
            for (let field of this.state.expressions) {
                saved[name].push([field.name, field.axis, field.color, field.function])
            }
            myStorage.setItem('savedFields', JSON.stringify(saved))
            this.$eventHub.$emit('presetsChanged')
        },

        resetModal () {
            this.name = ''
        },
        handleOk (bvModalEvt) {
            // Prevent modal from closing
            bvModalEvt.preventDefault()

            if (this.name.lenght > 0) {
                return
            }
            this.savePreset(this.name)

            // Hide the modal manually
            this.$nextTick(() => {
                this.$refs.modal.hide()
            })
        }
    },
    directives: {
        debounce
    }
}
</script>
<style>
/* MAIN */
    .plot-wrapper {
        max-height: 160px;
        overflow: hidden;
        overflow-y: scroll;
    }

/* COLOR PICKER */

    ul.colorpicker {
        font-family: 'Montserrat', sans-serif;
    }

    ul.colorpicker li {
        text-align: center;
        cursor: default;
        font-size: 13px;
        padding-top: 1px;
    }

    ul.colorpicker li:hover {
        background-color: #1E2536;
        border-left: 3px solid #1E2536;
    }

    ul.colorpicker li a {
        cursor: pointer;
    }

    li.field {
        line-height: 26px;
        padding-left: 20px;
        font-size: 90%;
    }

    li.plotsetup {
        display: block;
    }

    i {
        margin: 5px;
        padding: 0;
    }

    .plotname {
        display: inline-block;
        line-height: 15px;
        margin-bottom: 0;
        font-size: 13px;
        width: 59%;
        border: 1px solid grey;
        padding: 4.5px;
        border-radius: 20px;
    }

    .plotname:focus {
        background-color: rgba(241, 248, 255, 0.966);
        outline: none;
    }

    select {
        display: inline;
        border-radius: 5px;
        border: 1px solid rgb(156, 156, 156);
        background-color: rgb(255, 255, 255);
        padding: 2px 2.5px;
        color: #838282;
    }

    select:focus {
        border: 1.5px solid #d47f00;
        outline: none;
    }

    select option {
        background-color: rgb(216, 215, 215);
    }

    select option:hover {
        background-color: #d47f00;
    }

    .fa-trash {
        margin: 12px 5px 10px 1px !important;
        font-size: 10px;
        float: right;
    }

    .error {
        color: red;
    }
/* BUTTONS */

.btns-wrapper {
    display: flex;
    flex-flow: row wrap;
    justify-content: space-evenly;
    margin: 10px;
}

/* SAVE PRESET BUTTON */

    .save-preset {
        background-color:rgb(33, 41, 61);
        color: #fff;
        border-radius: 15px;
        padding: 0px 10px 0px 0px;
        border: 1px solid rgba(91, 100, 117, 0.76);
        font-size: 13px;
    }

    .save-preset:hover {
        background-color: rgb(47, 58, 87);
        box-shadow: 0px 0px 12px 0px rgba(37, 78, 133, 0.55);
        transition: all 0.5s ease;
    }

    .save-preset:focus {
        outline: none;
    }

/* ADD EXPRESSION BUTTON */

    .add-expression {
        background-color:rgb(33, 41, 61);
        color: #fff;
        border-radius: 15px;
        padding: 0px 10px 0px 0px;
        border: 1px solid rgba(91, 100, 117, 0.76);
        font-size: 13px;
    }

    .add-expression:hover {
        background-color: rgb(47, 58, 87);
        box-shadow: 0px 0px 12px 0px rgba(37, 78, 133, 0.55);
        transition: all 0.5s ease;
    }

    .add-expression:focus {
        outline: none;
    }

/* MEDIA QUERIES */

   @media (min-width: 1000px) and (max-width: 1440px) {
       p.plotname {
           width: 55%;
       }
   }

   @media (min-width: 2000px) {
       p.plotname {
           width: 60%;
       }
   }

</style>
