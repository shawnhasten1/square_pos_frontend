<template>
    <div id="receipt_body">
      <div class="titles">
        <h3>Current Sale</h3>
      </div>
      <ul class="receipt">
        <li class="line_item" v-for="item in items" :key="item" @click="editProduct(item)">
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
      </div>-
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
        <a href="#" class="btn btn_danger" @click="deleteKey()">Delete</a>
        <a href="#" class="btn btn_primary" @click="updateKey()">Update</a>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
export default {
    name: 'ReceiptBody',
    props: {
        total:Number,
        discount:Number,
        tax:Number,
        items:Object,
    },
    data: function(){
        return {
            edit_key:{
                "id": "",
                "quantity":0,
                "item_data": {
                    "category_id": "",
                    "name": ""
                }
            },
            receipt:{
                total:this.total,
                discount:this.discount,
                tax:this.tax,
                items:this.items
            },
        }
    },
    methods:{
        getTotals: function(){
            this.receipt.total = 0;
            this.receipt.tax = 0;
            for(var i = 0; i<this.receipt.items.length; i++){
                this.receipt.total+=this.receipt.items[i].item_data.variations[0].item_variation_data.price_money.amount*this.receipt.items[i].quantity;
            }
            this.receipt.tax = this.receipt.total*.065;
            this.receipt.total += this.receipt.tax;
        },
        charge: function(){
            var receipt_obj = {
                "line_items": [],
                "taxes": []
            }
            var taxes = [];
            for(var i = 0; i<this.receipt.items.length; i++){
                receipt_obj.line_items.push({
                "quantity": String(this.receipt.items[i].quantity),
                "catalog_object_id": this.receipt.items[i].item_data.variations[0].id,
                "item_type": "ITEM"
                })
                try{
                if(!taxes.includes(this.receipt.items[i].item_data.tax_ids[0])){
                    taxes.push(this.receipt.items[i].item_data.tax_ids[0])
                    receipt_obj.taxes.push({
                    "catalog_object_id": this.receipt.items[i].item_data.tax_ids[0]
                    })
                }
                }catch(error){
                console.log(error);
                }
            }
            console.log(receipt_obj)
            
            axios.post("http://localhost:5000/v1/orders", receipt_obj)
            .then(response => {
                console.log(response.data);
                this.receipt = {
                total:0,
                discount:0,
                tax:0,
                items:[]
                }
            })
        },
        editProduct: function(key){
            this.edit_key = key;
            this.modal.style.display = "block";
        },
        updateKey: function(){
            for(var i = 0; i<this.receipt.items.length; i++){
                if(this.receipt.items[i].id == this.edit_key.id){
                    this.receipt.items[i] = this.edit_key;
                }
            }
            this.modal.style.display = "none";
        },
        deleteKey: function(){
            var del_index = -1;
            for(var i = 0; i<this.receipt.items.length; i++){
                if(this.receipt.items[i].id == this.edit_key.id){
                    del_index = i;
                }
            }
            this.receipt.items.splice(del_index, 1);
            this.modal.style.display = "none";
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
    },
    updated: function () {
        this.getTotals()
    }
}
</script>