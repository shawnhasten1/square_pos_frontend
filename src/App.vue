<template>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Open+Sans:wght@300;400&display=swap" rel="stylesheet">
  <div id="container">
    <div id="categories">
      <div class="titles">
        <h3>Categories</h3>
      </div>
      <ul class="list">
        <li class="list_item" v-for="category in categories" :key="category" @click="refreshPriceKeys(category.id)" :class="{ active: category.id==category_id }">{{category.name}}</li>
      </ul>
    </div>
    <div id="pos_body">
      <div id="key_body" v-if="keys.length > 0 && !loading">
        <div class="pos_key" v-for="key in keys" :key="key" @click="getPriceKey(key.id)">
          <div class="key_image_holder">
            <img v-if="key.image_data.url" class="key_image" width="100%" v-bind:src="key.image_data.url"/>
            <img v-else class="key_image" width="100%" src="./assets/logo.png"/>
          </div>
          <div class="key_description">
            {{key.name}}<br/>${{formatMoney(key.price.amount/100)}}
          </div>
        </div>
      </div>
      <div id="key_body" v-else-if="loading">
        <h3 style="opacity: .5;">Loading...</h3>
      </div>
      <div id="key_body_nothing" v-else-if="!loading">
        <img class="nothing_image" width="100%" src="./assets/logo.png"/>
        <h3 style="text-align:center; opacity: .3;">Nothing Here</h3>
      </div>
    </div>
    <div id="receipt_body">
      <div class="titles">
        <h3>Current Sale</h3>
      </div>
      <ul class="receipt">
        <li class="line_item" v-for="item in receipt.items" :key="item" @click="editProduct(item)">
          <span style="display:inline-block">{{item.item_data.name}}</span>
          <span style="display:inline-block; float: right;">{{item.quantity}}x ${{formatMoney(item.item_data.variations[0].item_variation_data.price_money.amount/100)}}</span>
        </li>
      </ul>
      <div class="line_item">
          <span style="display:inline-block">Discounts</span>
          <span style="display:inline-block; float: right;">${{formatMoney(this.receipt.discount/100)}}</span>
      </div>
      <div class="line_item">
          <span style="display:inline-block">Tax</span>
          <span style="display:inline-block; float: right;">${{formatMoney(this.receipt.tax/100)}}</span>
      </div>
      <div class="charge_btn" @click="charge()">
        <div class="line_item">Charge ${{formatMoney(this.receipt.total/100)}}</div>
      </div>
    </div>
  </div>
  <!-- The Modal -->
  <div id="product_quantity" class="modal">
    <!-- Modal content -->
    <div class="modal-content">
      <div class="modal-header">
        <span class="close">&times;</span>
        <h2>{{edit_key.item_data.name}}</h2>
      </div>
      <div class="modal-body">
        <div class="quantity_btn minus_btn" @click="edit_key.quantity>1 && edit_key.quantity--">
          <span style="position: relative; left: 37px; top: -12px; font-size: 80px;">-</span>
        </div>
        <div>
          <h1>{{edit_key.quantity}}</h1>
        </div>
        <div class="quantity_btn plus_btn" @click="edit_key.quantity++">
          <span style="position: relative; left: 28px; top: -7px; font-size: 80px;">+</span>
        </div>
      </div>
      <div class="modal-footer">
        <a href="#" class="btn btn_primary" @click="updateKey()">Update</a>
      </div>
    </div>
  </div>
</template>

<script>

import axios from 'axios'

export default {
  name: 'App',
  data: function(){
    return {
        categories:[],
        keys:[],
        edit_key:{
            "id": "",
            "quantity":0,
            "item_data": {
                "category_id": "",
                "name": ""
            }
        },
        receipt:{
          total:0,
          discount:0,
          tax:0,
          items:[]
        },
        category_id:null,
        loading:false
      }
  },
  methods:{
    refreshCategories: function(){
      this.categories = []
      axios.get("http://localhost:5000/v1/categories")
        .then(response => {
          console.log(response.data);
          this.categories = response.data
        })
    },
    refreshPriceKeys: function(cat_id){
      this.keys = []
      this.category_id = cat_id
      this.loading = true
      axios.get("http://localhost:5000/v1/categories/"+this.category_id+"/items")
        .then(response => {
          console.log(response.data);
          this.keys = response.data
          this.loading = false
        })
    },
    getPriceKey: function(key_id){
      axios.get("http://localhost:5000/v1/items/"+key_id)
      .then(response => {
        console.log(response.data);
        var resp_data = response.data;
        resp_data['quantity'] = 1;
        this.receipt.items.push(resp_data);
        this.getTotals();
      })
    },
    editProduct: function(key){
      this.edit_key = key;
      this.modal.style.display = "block";
    },
    getTotals: function(){
        this.receipt.total = 0;
        this.receipt.tax = 0;
        for(var i = 0; i<this.receipt.items.length; i++){
          this.receipt.total+=this.receipt.items[i].item_data.variations[0].item_variation_data.price_money.amount*this.receipt.items[i].quantity;
        }
        this.receipt.tax = this.receipt.total*.065;
        this.receipt.total += this.receipt.tax;
    },
    updateKey: function(){
      for(var i = 0; i<this.receipt.items.length; i++){
        if(this.receipt.items[i].id == this.edit_key.id){
          this.receipt.items[i] = this.edit_key;
        }
      }
      this.modal.style.display = "none";
      this.getTotals();
    },
    charge: function(){
      var receipt_obj = {
        "line_items": [],
        "taxes": []
      }
      for(var i = 0; i<this.receipt.items.length; i++){
        receipt_obj.line_items.push({
          "quantity": this.receipt.items[i].quantity,
          "catalog_object_id": this.receipt.items[i].item_data.variations[0].id,
          "item_type": "ITEM"
        })
        try{
          receipt_obj.taxes.push({
            "catalog_object_id": this.receipt.items[i].item_data.tax_ids[0]
          })
        }catch(error){
          console.log(error);
        }
      }
      console.log(receipt_obj)
      
      axios.post("http://localhost:5000/v1/orders", receipt_obj)
      .then(response => {
        console.log(response.data);
         this.listings = response.data
      })
    },
    formatMoney: function(amount, decimalCount = 2, decimal = ".", thousands = ",") {
        try {
            decimalCount = Math.abs(decimalCount);
            decimalCount = isNaN(decimalCount) ? 2 : decimalCount;

            const negativeSign = amount < 0 ? "-" : "";

            let i = parseInt(amount = Math.abs(Number(amount) || 0).toFixed(decimalCount)).toString();
            let j = (i.length > 3) ? i.length % 3 : 0;

            return negativeSign + (j ? i.substr(0, j) + thousands : '') + i.substr(j).replace(/(\d{3})(?=\d)/g, "$1" + thousands) + (decimalCount ? decimal + Math.abs(amount - i).toFixed(decimalCount).slice(2) : "");
        } catch (e) {
            //console.log(e)
        }
    }
  },
  mounted() {
    this.refreshCategories();
    this.modal = document.getElementById("product_quantity");

    // Get the <span> element that closes the modal
    var span = document.getElementsByClassName("close")[0];

    // When the user clicks on <span> (x), close the modal
    span.onclick = function() {
      this.modal.style.display = "none";
    }

    // When the user clicks anywhere outside of the modal, close it
    window.onclick = function(event) {
      if (event.target == this.modal) {
        this.modal.style.display = "none";
      }
    }
  }
}
</script>

<style>
  body{
    margin: 0;
    padding: 0;
    font-family: 'Open Sans', sans-serif;
  }
  #container{
    display: flex;
    flex-wrap: nowrap;
    margin: 0;
    padding: 0;
  }
  #categories{
    width: 20vw;
    height: 100vh;
    background: white;
    display: flex;
    flex-direction: column;
    align-items: center;
    border-right: solid 1px rgb(229 229 229 / 52%);
  }
  .titles{    
    width: 100%;
    text-align: center;
    background: black;
    color: white;
  }
  .list{
    margin: 0;
    padding: 0;
    list-style: none;
    width: 100%;
  }
  .list_item{
    padding: 1.25rem;
    transition: .3s;
    box-shadow: 0px 5px 5px 0px rgba(212,212,212,0.75);
    -webkit-box-shadow: 0px 5px 5px 0px rgba(212,212,212,0.75);
    -moz-box-shadow: 0px 5px 5px 0px rgba(212,212,212,0.75);
    font-weight: 600;
  }
  .list_item:hover, .list_item.active{
    background: grey;
    color: white;
    cursor: pointer;
  }
  .list_item:active, .list_item.active:active{
    background: rgb(80, 80, 80);
  }
  #pos_body{
    width: 55vw;
    height: 100vh;
    background: white;
    border-right: solid 1px rgb(229 229 229 / 52%);
  }
  #key_body{
    padding: 1rem;
    display: inline-block;
  }
  .pos_key{
    background: white;
    box-shadow: 0px 10px 5px 0px rgba(212,212,212,0.75);
    -webkit-box-shadow: 0px 10px 5px 0px rgba(212,212,212,0.75);
    -moz-box-shadow: 0px 10px 5px 0px rgba(212,212,212,0.75);
    border-radius: 3px;
    width: 200px;
    height: 200px;
    margin: 10px;
    display: inline-block;
    transition: .3s;
  }
  .pos_key:hover{
    box-shadow: 0px 5px 5px 0px rgba(212,212,212,0.75);
    -webkit-box-shadow: 0px 5px 5px 0px rgba(212,212,212,0.75);
    -moz-box-shadow: 0px 5px 5px 0px rgba(212,212,212,0.75);
    opacity: .8;
    cursor: pointer;
  }
  .key_image_holder{
    border-top-left-radius: 3px;
    border-top-right-radius: 3px;
    height: 150px;
    overflow: hidden;
  }
  .key_image{
    transition:.3s;
    width: auto;
    min-width: 100%;
    height: 100%;
  }
  .pos_key:hover .key_image{
    transform: scale(1.1);
  }
  .key_description{
    border-top: 1px solid rgba(212,212,212,0.75);
    text-align: center;
  }
  .key_body_nothing{
    display: block;
    margin: 0 auto;
  }
  .nothing_image{
    width: 100%;
    max-width: 250px;
    display: block;
    margin: 0 auto;
    margin-top:25%;
    filter: none;
    -webkit-filter: grayscale(100%);
    -moz-filter:    grayscale(100%);
    -ms-filter:     grayscale(100%);
    -o-filter:      grayscale(100%);
    opacity: .3;
  }
  #receipt_body{
    width: 25vw;
    height: 100vh;
    background: white;
    display: flex;
    flex-direction: column;
    border-right: solid 1px rgb(229 229 229 / 52%);
  }
  .receipt{
    margin: 0;
    padding: 0;
    list-style: none;
    width: 100%;
    border-bottom:solid 1px rgb(229 229 229 / 52%);
    min-height: 55px;
  }
  .receipt > .line_item:hover{
    cursor: pointer;
    opacity: .8;
    transition: .3s;
  }
  .receipt > .line_item:nth-child(even){
   background: grey;
   color: white;
  }
  .line_item{
    padding: 1.25rem;
    padding-top: 1rem;
    padding-bottom: 1rem;
    font-weight: 600;
  }
  .charge_btn{
    margin-top: auto;
    color: white;
    background: rgb(41, 131, 248);
    text-align: center;
  }
  .charge_btn:hover{
    opacity: .8;
    transition: .3s;
    cursor: pointer;
  }

  .btn{
    padding: .5rem;
    border-radius: 3px;
    color: white;
    text-decoration: none;
  }
  .btn:hover{
    cursor: pointer;
  }
  .btn_primary{
    background: rgb(41, 131, 248);
  }
  .btn_primary:hover{
    background: rgb(23, 95, 189);
  }
  .btn_primary:active{
    background: rgb(20, 80, 160);
  }

  .quantity_btn{
    width:100px;
    height:100px;
    border-radius:50px;
    border:solid 1px rgba(212,212,212,0.75);
    position:relative;
    margin-left: 20px;
    margin-right: 20px;
    transition: .3s;
    -webkit-touch-callout: none; /* iOS Safari */
      -webkit-user-select: none; /* Safari */
      -khtml-user-select: none; /* Konqueror HTML */
        -moz-user-select: none; /* Old versions of Firefox */
          -ms-user-select: none; /* Internet Explorer/Edge */
              user-select: none; /* Non-prefixed version, currently
                                    supported by Chrome, Edge, Opera and Firefox */
  }
  .minus_btn:hover{
    background: red;
    color: white;
    cursor: pointer;
  }
  .plus_btn:hover{
    background: green;
    color: white;
    cursor: pointer;
  }

  /*MODALS*/
  .modal {
    display: none; /* Hidden by default */
    position: fixed; /* Stay in place */
    z-index: 1; /* Sit on top */
    padding-top: 100px; /* Location of the box */
    left: 0;
    top: 0;
    width: 100%; /* Full width */
    height: 100%; /* Full height */
    overflow: auto; /* Enable scroll if needed */
    background-color: rgb(0,0,0); /* Fallback color */
    background-color: rgba(0,0,0,0.4); /* Black w/ opacity */
  }

  /* Modal Content */
  .modal-content {
    position: relative;
    background-color: #fefefe;
    margin: auto;
    padding: 0;
    border: 1px solid #888;
    width: 500px;
    box-shadow: 0 4px 8px 0 rgba(0,0,0,0.2),0 6px 20px 0 rgba(0,0,0,0.19);
    -webkit-animation-name: animatetop;
    -webkit-animation-duration: 0.4s;
    animation-name: animatetop;
    animation-duration: 0.4s
  }

  /* Add Animation */
  @-webkit-keyframes animatetop {
    from {top:-300px; opacity:0} 
    to {top:0; opacity:1}
  }

  @keyframes animatetop {
    from {top:-300px; opacity:0}
    to {top:0; opacity:1}
  }

  /* The Close Button */
  .close {
    float: right;
    font-size: 28px;
    font-weight: bold;
  }

  .close:hover,
  .close:focus {
    color: #000;
    text-decoration: none;
    cursor: pointer;
  }

  .modal-header {
    padding: 2px 16px;
    border-bottom: 1px solid rgba(212,212,212,0.75);
  }

  .modal-body {    
    padding: 2px 16px;
    display: flex;
    flex-direction: row;
    align-content: center;
    align-items: center;
    justify-content: center;
  }

  .modal-footer {
    border-top: 1px solid rgba(212,212,212,0.75);
    padding: 16px 16px;
    display: flex;
    flex-direction: row-reverse;
  }
</style>
