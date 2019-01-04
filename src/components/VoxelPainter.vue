<template>
  <div id="painter">
    {{ title }}
    <div id="buttons">
      <el-button type="success" @click="togglePlant" >New small Plant</el-button>
      <el-button type="success" @click="toggleBigPlant">New Big Plant</el-button>
      <el-button type="warning" @click="toggleEdit" plain>Edit Plants</el-button>
      <el-button type="danger" @click="toggleDelete" plain>Delete Plants</el-button>
		</div>
    <div id="dimensions">
      <el-row :gutter="20">
        <el-col :span="10" >
          <div class="grid-content">
            <h3>Room X</h3>
          </div>
        </el-col>
        <el-col :span="10">
          <div class="grid-content">
            <el-input-number v-model="roomX" :min="20" :max="1000" :step="10" @change="refreshRoom"></el-input-number>
          </div>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :span="10">
          <div class="grid-content">
            <h3>Room Y</h3>
          </div>
        </el-col>
        <el-col :span="10">
          <div class="grid-content">
            <el-input-number v-model="roomY" :min="20" :max="1000" :step="10" @change="refreshRoom"></el-input-number>
          </div>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :span="10">
          <div class="grid-content">
            <h3>Grid Size</h3>
          </div>
        </el-col>
        <el-col :span="10">
          <div class="grid-content">
            <el-input-number v-model="gridSize" :min="5" :max="200" :step="5" @change="refreshRoom"></el-input-number>
          </div>
        </el-col>
      </el-row>
    </div>
    <div id="canvas"></div>
  </div>
</template>

<script src="../assets/threex.domevents.js"></script>
<script>
import * as THREE from 'three'
// import * as DE from 'threex.domevents'
import GLTFLoader from 'three-gltf-loader'
import OrbitControls from 'three-orbitcontrols'
import TextSprite from 'three.textsprite'

import THREEx from '../assets/threex.domevents.js'

// var THREE = require('three')
// var initializeDomEvents = require('threex.domevents')
// var THREEx = {}
// initializeDomEvents(THREE, THREEx);

export default {
  name: 'VoxelPainter',
  props: {
    title: String
  },
  data: function() {
    return {
      height: 600,
      roomX: 240,
      roomY: 240,
      gridSize: 20,

      small_plant_active: false,
      big_plant_active: false,
      plant_edit: false,
      plant_delete: false,
      camera: 0,
      scene: 0,
      gridHelper: 0,

      rollOverGeo: 0,
      rollOverMesh: 0,
      rollOverMaterial: 0,

      rollOverGeoBig: 0,
      rollOverMeshBig: 0,
      rollOverMaterialBig: 0,

      renderer: 0,
      controls: 0,
      domEvent: 0,
      ambientLight: 0,
      directionalLight: 0,

      loader: 0,
      loader_big: 0,

      objects: [],
      existingPlants: [
        {name: 'plant', type:'big', scale: 1.4, x:50, y:0, z:50},
        {name: 'plant2', type:'small', scale: 1.4, x:10, y:0, z:10}
      ],
      loadedObjects: [],

      raycaster: 0,
      mouse: false,

      geometry: 0,
      plane: 0,

      isShiftDown: false,

      plant: 0,
      activePlant: 0,
      editPlantName: '',
      selected: 0,

      // buffer for all plants (whole object)
      globalPlants: [],

      boundingBoxMesh: 0,
      boundingBoxGeo: 0,
      boundingBoxMaterial: 0,

      topText: 0,
      bottomText: 0,
      leftText: 0,
      rightText: 0,
    }

  },
  methods: {
    togglePlant: function() {
      if (this.small_plant_active == true) {
        this.small_plant_active = false;
        this.rollOverMaterial.visible = false;
      } else {
        this.small_plant_active = true;
        this.big_plant_active = false;
      }
    },
    toggleBigPlant: function() {
      if (this.big_plant_active == true) {
        this.big_plant_active = false;
        this.rollOverMaterialBig.visible = false;
      } else {
        this.big_plant_active = true;
        this.small_plant_active = false;
      }
    },
    toggleEdit: function() {
      if (this.plant_edit == true) {
        this.plant_edit = false;
      } else {
        this.plant_edit = true;
      }
    },
    toggleDelete: function() {
      if (this.plant_delete == true) {
        this.plant_delete = false;
      } else {
        this.plant_delete = true;
      }
    },
    openEditModal: function (e, selected) {
      this.$prompt('Insert new plant name', 'Edit plant ' + selected.name, {
          confirmButtonText: 'OK',
        //  cancelButtonText: 'Cancel',
          // inputPattern: '',
          inputErrorMessage: 'Invalid plant name'
        }).then(({ value }) => {
          selected.name = value;

          selected.remove(selected.children[1])
          console.log(selected)

          this.plantText = new TextSprite({
          	textSize: 4,
          	texture: {
          		text: selected.name,
          		fontFamily: 'Ubuntu, Helvetica, sans-serif',
          },
          	material: {color: 0x45903c},
          });

          this.plantText.position.set(0, 0, this.gridSize)
          this.selected.add(this.plantText)

          this.$message({
            type: 'success',
            message: 'New plant name: ' + value
          });
          console.log(this.globalPlants)
        }).catch(() => {
          this.$message({
            type: 'info',
            message: 'Input canceled'
          });
        });
    },
    openNewModal: function () {
      this.$prompt('Please select plant name', 'New plant', {
          confirmButtonText: 'OK',
          inputErrorMessage: 'Invalid plant name'
        }).then(({ value }) => {
          this.plant.name = value;
          console.log(this.plant)
          this.plantText = new TextSprite({
          	textSize: 4,
          	texture: {
          		text: this.plant.name,
          		fontFamily: 'Ubuntu, Helvetica, sans-serif',
          },
          	material: {color: 0x45903c},
          });


          if (this.plant.type == 'small') {
            this.plantText.position.set(0, 0, this.gridSize)
            this.plant.add(this.plantText)

            this.boundingBoxGeo = new THREE.BoxGeometry(this.gridSize /1.5 , this.gridSize /1.5, this.gridSize /1.5 );
        		this.boundingBoxMaterial = new THREE.MeshBasicMaterial( { visible: false, color: 0x454e5a, opacity: 0.5, transparent: true } );
        		this.boundingBoxMesh = new THREE.Mesh( this.boundingBoxGeo, this.boundingBoxMaterial )

            this.boundingBoxMesh.position.z = this.gridSize /4;

            this.plant.add(this.boundingBoxMesh)
            this.globalPlants.push(this.plant)

            this.scene.add( this.plant );

          } else {
            this.plantText.position.set(0, 0, this.gridSize * 2.4)
            this.plant.add(this.plantText)

            this.boundingBoxGeo = new THREE.BoxGeometry(this.gridSize , this.gridSize, this.gridSize *2 );
        		this.boundingBoxMaterial = new THREE.MeshBasicMaterial( { visible: false, color: 0x454e5a, opacity: 0.5, transparent: true } );
        		this.boundingBoxMesh = new THREE.Mesh( this.boundingBoxGeo, this.boundingBoxMaterial )

            this.boundingBoxMesh.position.z = this.gridSize;

            this.plant.add(this.boundingBoxMesh)
            this.globalPlants.push(this.plant)

            this.scene.add( this.plant );


          }

          // this.scene.add( this.plant );
          this.objects.push( this.plant );
          // let selected = this.plant
          //
          // this.domEvent.addEventListener(this.plant, 'click', (event, selected)  => {
          //   console.log(selected)
          //   if (this.plant_edit == true) {
          //     this.openEditModal(event, this.plant);
          //   }
          // }, false);

          this.$message({
            type: 'success',
            message: 'New plant: ' + value
          });
        }).catch(() => {
          this.$message({
            type: 'info',
            message: 'Input canceled'
          });
        });
        // this.loop();
    },
    refreshRoom: function() {
      this.scene.remove(this.plane);
      this.scene.remove(this.gridHelper);
      this.addPlane();
      this.createGridHelper();

      // we also need to refresh the size of the rollOverMesh
      this.scene.remove(this.rollOverMesh);
      this.scene.remove(this.rollOverMeshBig);
      this.createRollOverGeo();

      // remove and refresh the textTop etc.
      this.scene.remove(this.topText);
      this.scene.remove(this.bottomText);
      this.scene.remove(this.leftText);
      this.scene.remove(this.rightText);
      this.addText();

      this.controls.target.set(this.roomX/2,0,this.roomY/2);
      this.controls.update();
    },
    addScene: function() {
      this.scene = new THREE.Scene();
  		this.scene.background = new THREE.Color( 0x3e4653 );

      this.camera = new THREE.PerspectiveCamera( 45, window.innerWidth / this.height, 0.1, 10000 );
  		this.camera.position.set( 500, 800, 2300 )

      // lights
      this.ambientLight = new THREE.AmbientLight( 0x606060 );
      this.scene.add( this.ambientLight );

      this.directionalLight = new THREE.DirectionalLight( 0xffffff );
      this.directionalLight.position.set( 1, 0.75, 0.5 ).normalize();
      this.scene.add( this.directionalLight );
    },
    createRollOverGeo: function() {
      this.rollOverGeo = new THREE.BoxBufferGeometry( this.gridSize, this.gridSize, this.gridSize );
  		this.rollOverMaterial = new THREE.MeshBasicMaterial( { visible: false, color: 0x454e5a, opacity: 0.5, transparent: true } );
      // this.rollOverMaterialRed = new THREE.MeshBasicMaterial( { visible: false, color: 0xf56c6c, opacity: 0.5, transparent: true } );
  		this.rollOverMesh = new THREE.Mesh( this.rollOverGeo, this.rollOverMaterial )
  		this.scene.add(this.rollOverMesh)

      this.rollOverGeoBig = new THREE.BoxBufferGeometry( this.gridSize*2, this.gridSize*2, this.gridSize*2 );
  		this.rollOverMaterialBig = new THREE.MeshBasicMaterial( { visible: false, color: 0x454e5a, opacity: 0.5, transparent: true } );
  		this.rollOverMeshBig = new THREE.Mesh( this.rollOverGeoBig, this.rollOverMaterialBig );
  		this.scene.add(this.rollOverMeshBig)
    },
    createGridHelper: function() {
      if (this.roomX >= this.roomY) {
        this.gridHelper = new THREE.GridHelper( this.roomX * 2, this.roomX/this.gridSize * 2,  0x3e4653, 0x3e4653 );
        this.scene.add( this.gridHelper );
      } else {
        this.gridHelper = new THREE.GridHelper( this.roomY * 2, this.roomY/this.gridSize * 2,  0x3e4653, 0x3e4653 );
        this.scene.add( this.gridHelper );
      }

    },
    addPlane: function() {
      // raycaster
  		this.raycaster = new THREE.Raycaster();
  		this.mouse = new THREE.Vector2();

  		this.geometry = new THREE.PlaneBufferGeometry( this.roomX , this.roomY );
  		this.geometry.rotateX( - Math.PI / 2 );

  		this.plane = new THREE.Mesh( this.geometry, new THREE.MeshBasicMaterial( { visible: true, color: 0xfeb74c  } ) );
  		this.plane.position.x = this.roomX/2;
  		this.plane.position.z = this.roomY/2;

  		//plane.position.set(gridSize,0,gridSize);
  		this.scene.add( this.plane );

  		this.objects.push( this.plane );
    },
    addText: function() {
      // Load Sprite
  		this.topText = new TextSprite({
  			textSize: 10,
  			texture: {
      		text: 'Top',
      		fontFamily: 'Ubuntu, Helvetica, sans-serif',
  		},
  			material: {color: 0xfeb74c},
  		});
  		this.topText.position.set(this.roomX/2 , 15, -50);
  		this.scene.add(this.topText)

      this.bottomText = new TextSprite({
  			textSize: 10,
  			texture: {
      		text: 'Bottom',
      		fontFamily: 'Ubuntu, Helvetica, sans-serif',
  		},
  			material: {color: 0xfeb74c},
  		});
  		this.bottomText.position.set(this.roomX/2 , 15, this.roomY+50);
  		this.scene.add(this.bottomText)

  		this.leftText = new TextSprite({
  			textSize: 10,
  			texture: {
      		text: 'Left',
      		fontFamily: 'Ubuntu, Helvetica, sans-serif',
  		},
  			material: {color: 0xfeb74c},
  		});
  		this.leftText.position.set(-50 , 15, this.roomY/2);
  		this.scene.add(this.leftText)

  		this.rightText = new TextSprite({
  			textSize: 10,
  			texture: {
      		text: 'Right',
      		fontFamily: 'Ubuntu, Helvetica, sans-serif',
  		},
  			material: {color: 0xfeb74c},
  		});
  		this.rightText.position.set(this.roomX + 50 , 15, this.roomY/2);
  		this.scene.add(this.rightText)
    },
    loadPlants: function() {
      for (var i=0; i<this.existingPlants.length; i++) {
        if (this.existingPlants[i].type == 'small') {
          let actual = this.existingPlants[i]
          this.loader.load( 'scene.gltf', ( gltf ) => {
						this.plant = gltf.scene.children[0];
						this.plant.position.set(actual.x,actual.y, actual.z);
						this.plant.scale.set(actual.scale, actual.scale, actual.scale);

            this.plant.name = actual.name

						// this.domEvent.addEventListener(this.plant, 'click', (event)  => {
            //   if (this.plant_edit == true) {
            //     this.openModal();
            //   }
            // }, false);

            // Load Sprite as Plant Description
            this.plantText = new TextSprite({
            	textSize: 5,
            	texture: {
            		text: actual.name,
            		fontFamily: 'Ubuntu, Helvetica, sans-serif',
            },
            	material: {color: 0x45903c},
            });
            this.plantText.position.set(0, 0, this.gridSize);
            // this.activePlant = new THREE.Box3().setFromObject(this.plant);

            // this.activePlant.add(this.plantText)
            this.plant.add(this.plantText)

            this.boundingBoxGeo = new THREE.BoxGeometry(this.gridSize / 1.5 , this.gridSize / 1.5, this.gridSize /1.5 );
        		this.boundingBoxMaterial = new THREE.MeshBasicMaterial( { visible: false, color: 0x454e5a, opacity: 0.5, transparent: true } );
        		this.boundingBoxMesh = new THREE.Mesh( this.boundingBoxGeo, this.boundingBoxMaterial )

            //this.boundingBoxMesh.position.copy( this.plant.position );
  					// this.boundingBoxMesh.position.divideScalar( this.gridSize ).floor().multiplyScalar( this.gridSize ).addScalar( this.gridSize / 2 );
            this.boundingBoxMesh.position.z = this.gridSize / 4 ;

            this.plant.add(this.boundingBoxMesh)
            // this.plant.add(this.activePlant)
            this.globalPlants.push(this.plant)

            this.scene.add( this.plant );

						// this.objects.push( this.plant );
            this.objects.push(this.boundingBoxMesh)

	        });
        }
        if (this.existingPlants[i].type == 'big') {
          let actual = this.existingPlants[i]
          this.loader_big.load( 'scene.gltf', ( gltf ) => {
						this.plant = gltf.scene.children[0];
            // this.plant.position.x = this.existingPlants[i].x
						this.plant.position.set(actual.x,actual.y, actual.z);
						this.plant.scale.set(actual.scale, actual.scale, actual.scale);

            this.plant.name = actual.name

						// this.domEvent.addEventListener(this.plant, 'click', (event)  => {
            //   if (this.plant_edit == true) {
            //     this.openModal();
            //   }
            // }, false);

            // Load Sprite as Plant Description
            this.plantText = new TextSprite({
            	textSize: 5,
            	texture: {
            		text: actual.name,
            		fontFamily: 'Ubuntu, Helvetica, sans-serif',
            },
            	material: {color: 0x45903c},
            });
            this.plantText.position.set(0, 0, this.gridSize * 2.4);
            this.plant.add(this.plantText)

            this.boundingBoxGeo = new THREE.BoxGeometry(this.gridSize , this.gridSize, this.gridSize *2 );
        		this.boundingBoxMaterial = new THREE.MeshBasicMaterial( { visible: false, color: 0x454e5a, opacity: 0.5, transparent: true } );
        		this.boundingBoxMesh = new THREE.Mesh( this.boundingBoxGeo, this.boundingBoxMaterial )

            this.boundingBoxMesh.position.z = this.gridSize  ;

            this.plant.add(this.boundingBoxMesh)
            this.globalPlants.push(this.plant)

            this.scene.add( this.plant );
						// this.objects.push( this.plant );
            this.objects.push(this.boundingBoxMesh)

	        });
        }
      }
    },
    onWindowResize: function() {
			this.camera.aspect = window.innerWidth / window.innerHeight;
			this.camera.updateProjectionMatrix();

			this.renderer.setSize( window.innerWidth, window.innerHeight );
		},
    onDocumentMouseMove: function( event ) {

			// event.preventDefault();
			// let sp = document.getElementById('small_plant').classList.contains('active')
			// let bp = document.getElementById('big_plant').classList.contains('active')
			if ( this.small_plant_active == true) {

				this.mouse.set( ( event.clientX / window.innerWidth ) * 2 - 1, - ( event.clientY / window.innerHeight ) * 2 + 1 );

				this.raycaster.setFromCamera( this.mouse, this.camera );
				var intersects = this.raycaster.intersectObjects( this.objects );

				if ( intersects.length > 0 ) {
					var intersect = intersects[ 0 ];
					this.rollOverMaterial.visible = true;

					this.rollOverMesh.position.copy( intersect.point ).add( intersect.face.normal );
					this.rollOverMesh.position.divideScalar( this.gridSize ).floor().multiplyScalar( this.gridSize ).addScalar( this.gridSize / 2 );
          // console.log(this.gridSize);
				}
				//this.loop();
			}
			if ( this.big_plant_active == true) {

				this.mouse.set( ( event.clientX / window.innerWidth ) * 2 - 1, - ( event.clientY / window.innerHeight ) * 2 + 1 );

				this.raycaster.setFromCamera( this.mouse, this.camera );
				var intersects = this.raycaster.intersectObjects( this.objects );

				if ( intersects.length > 0 ) {
					var intersect = intersects[ 0 ];
					this.rollOverMaterialBig.visible = true;
					this.rollOverMeshBig.position.copy( intersect.point ).add( intersect.face.normal );
					this.rollOverMeshBig.position.divideScalar( this.gridSize*2 ).floor().multiplyScalar( this.gridSize *2 ).addScalar( this.gridSize );
				}
				// this.loop();
			}
      if ( this.plant_delete == true) {
        this.mouse.set( ( event.clientX / window.innerWidth ) * 2 - 1, - ( event.clientY / window.innerHeight ) * 2 + 1 );

				this.raycaster.setFromCamera( this.mouse, this.camera );
				var intersects = this.raycaster.intersectObjects( this.objects );

				if ( intersects.length > 0 ) {
					var intersect = intersects[ 0 ];
					this.rollOverMaterial.visible = true;
          this.rollOverMesh.material.color.setHex(0xf56c6c);
          // this.rollOverMaterial.color = '0xf56c6c';

					this.rollOverMesh.position.copy( intersect.point ).add( intersect.face.normal );
					this.rollOverMesh.position.divideScalar( this.gridSize ).floor().multiplyScalar( this.gridSize ).addScalar( this.gridSize / 2 );
          // console.log(this.gridSize);
				}
      }
		},
    onDocumentMouseDown: function( event ) {
      // this condition should improve speed
      if (this.small_plant_active == true || this.big_plant_active == true || this.plant_edit == true || this.plant_delete) {
        this.mouse.set( ( event.clientX / window.innerWidth ) * 2 - 1, - ( event.clientY / window.innerHeight ) * 2 + 1 );
        // this.mouse.set(event.clientX, event.clientY)
  			this.raycaster.setFromCamera( this.mouse, this.camera );

        // for (let i=0; i<=this.objects.length; i++) {
        //   let bbox = new THREE.Box3.setFromObject(this.objects.length[i])
        //   this.boundingBoxes.push(bbox)
        // }

  			var intersects = this.raycaster.intersectObjects( this.objects, true );
        if ( intersects.length > 0 ) {
  				var intersect = intersects[ 0 ];

  				// delete cube, not working right now, as we add a scene and not a mesh to the main scene
  				// if ( this.isShiftDown ) {
  				// 	if ( intersect.object.parent !== this.scene ) {
  				// 		this.objects.splice( this.objects.indexOf( intersect.object ), 1 );
  				// 		intersect.object.parent.remove(intersect.object);
  				// 	}
  				// }
          if (this.small_plant_active == true) {
            this.small_plant_active = false;
            this.rollOverMaterial.visible = false;
  					this.loader.load( 'scene.gltf', ( gltf ) => {
    					this.plant = gltf.scene.children[0];

    					this.plant.position.copy(intersect.point).add(intersect.face.normal);
    					this.plant.position.divideScalar(this.gridSize).floor().multiplyScalar(this.gridSize).addScalar(this.gridSize/2);
    					this.plant.scale.set(this.gridSize/100 * 7,this.gridSize/100 * 7,this.gridSize/100 * 7);
    					this.plant.position.y = 0;

              this.plant.type = 'small';



  					// this.plant.name = 'plant_new';
  					// this.plant.strain = 'purple';
            //
          	// this.scene.add( this.plant );
  					// this.objects.push( this.plant );
  					// this.domEvent.addEventListener(this.plant, 'click', (event)  => {
            //   if (this.plant_edit == true) {
            //     this.openModal();
            //   }
            // }, false);

            // Load Sprite as Plant Description
            // this.plantText = new TextSprite({
            // 	textSize: 5,
            // 	texture: {
            // 		text: this.plant.name,
            // 		fontFamily: 'Ubuntu, Helvetica, sans-serif',
            // },
            // 	material: {color: 0x45903c},
            // });
            // this.plantText.position.set(this.plant.position.x , this.plant.position.y + this.plant.scale.y*20, this.plant.position.z);
            // this.scene.add(this.plantText)
  	        });
            this.openNewModal(event);
  				}
  				if (this.big_plant_active == true) {

            this.big_plant_active = false;
            this.rollOverMaterialBig.visible = false;

  					this.loader_big.load( 'scene.gltf', ( gltf ) => {
  						this.plant = gltf.scene.children[0];
  						this.plant.position.copy(intersect.point).add(intersect.face.normal);
  						this.plant.position.divideScalar(this.gridSize*2).floor().multiplyScalar(this.gridSize*2).addScalar(this.gridSize);
  						this.plant.scale.set(this.gridSize/100 * 7,this.gridSize/100 * 7,this.gridSize/100 * 7);
  						this.plant.position.y = 0;

              this.plant.type = 'big';

  						// this.plant.name = 'plant_big';
  						// this.plant.strain = 'purple_big';
              //
  	        	// this.scene.add( this.plant );
  						// this.objects.push( this.plant );
              //
              // // Load Sprite as Plant Description
              // this.plantText = new TextSprite({
              // 	textSize: 5,
              // 	texture: {
              // 		text: this.plant.name,
              // 		fontFamily: 'Ubuntu, Helvetica, sans-serif',
              // },
              // 	material: {color: 0x45903c},
              // });
              // this.plantText.position.set(this.plant.position.x , this.plant.position.y + this.plant.scale.y*48, this.plant.position.z);
              // this.scene.add(this.plantText)

  						// domEvents.addEventListener(plant, 'dblclick', event => {
  						// 	window.alert(plant)
  						// });
  						// domEvents.addEventListener(plant, 'click', event => {
  						// 	let editbutton = document.getElementById('edit_plant').classList.contains('active');
  						// 	if (editbutton == true) {
  						// 		window.alert(plant)
  						// 	}
  						// })
  	        });
            this.openNewModal(event);
  				}
          // console.log(intersect.object)
          if (this.plant_delete == true) {

            if ( intersect.object.parent !== this.scene ) {
              this.rollOverMaterial.visible = false
              this.rollOverMesh.material.color.setHex(0x454e5a);
              this.plant_delete = false
              // this is the plant with the label
              // console.log(intersect.object.parent.parent.parent)

              this.objects.splice( this.objects.indexOf( intersect.object ), 1 );
              this.globalPlants.splice( this.globalPlants.indexOf(intersect.object.parent), 1)
              // intersect.object.parent.parent.parent.parent.remove(intersect.object.parent.parent.parent)
              // this is the whole object
              // console.log(intersect.object.parent)
              // intersect.object.parent.remove(intersect.object.children)
              intersect.object.parent.parent.remove(intersect.object.parent)
              // this.scene.remove(intersect.object.parent)
  					}
          }
          if (this.plant_edit == true) {
            if ( intersect.object.parent !== this.scene) {
              this.plant_edit = false
              this.selected = this.scene.getObjectByName(intersect.object.parent.name)
              this.openEditModal(event, this.selected)
            }
          }
  			}
      }
			// event.preventDefault();



		},
    initLoaders: function() {
      this.loader = new GLTFLoader().setPath('/small_plant/');
      this.loader_big = new GLTFLoader().setPath('/big_plant/');
    },
    init: function() {
      this.initLoaders();
      this.addScene();
      // roll-over helpers
      this.createRollOverGeo();
      this.createGridHelper();

      this.addPlane();
      this.addText();

      this.loadPlants();
      this.renderer = new THREE.WebGLRenderer( { antialias: true } );
  		this.renderer.setPixelRatio( window.innerWidth / this.height );
  		this.renderer.setSize( window.innerWidth , this.height  );

      let canvas = document.getElementById('canvas')
  		canvas.appendChild( this.renderer.domElement );

      this.controls = new THREE.OrbitControls(this.camera, this.renderer.domElement);
  		this.controls.maxDistance = this.roomX*3;
  		this.controls.minDistance = this.gridSize;
  		this.controls.target.set(this.roomX/2,0,this.roomY/2)
  		this.controls.update()

      // var THREEx = {}
      // DE(THREE, THREEx)

      this.domEvent	= new THREEx.DomEvents(this.camera, this.renderer.domElement);

  		document.addEventListener( 'mousemove', this.onDocumentMouseMove, false );
  		document.addEventListener( 'click', this.onDocumentMouseDown, false );
  		document.addEventListener( 'keydown', this.onDocumentKeyDown, false );
  		document.addEventListener( 'keyup', this.onDocumentKeyUp, false );

  		window.addEventListener( 'resize', this.onWindowResize, false );
    },
    loop: function () {
      requestAnimationFrame(this.loop);
      this.controls.update();
      this.render();
    },
    render: function() {
      this.renderer.render( this.scene, this.camera );
    }
  },
  mounted: function() {


    this.init();

    this.loop();

    // function onDocumentKeyDown( event ) {
		// 	switch ( event.keyCode ) {
		// 		case 16: isShiftDown = true; break;
		// 	}
		// }
    //
		// function onDocumentKeyUp( event ) {
		// 	switch ( event.keyCode ) {
		// 		case 16: isShiftDown = false; break;
		// 	}
		// }
	}
}

</script>

<style scoped>
#buttons {
	position: relative;
	top: 80px;
	padding: 5px;
	font-size:13px;
	text-align:center;
}
#dimensions {
	position: absolute;
	top: 225px;
	right: 60px;
	padding: 5px;
	font-size:13px;
  color: white;

}
#canvas {
  width: 100%;
  height: 600px;
}
</style>
