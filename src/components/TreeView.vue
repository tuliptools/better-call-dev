<template>
  <div :class="parsedData.klass">
    <TreeViewItem
      class="tree-view-item-root"
      :data="parsedData"
      :max-depth="maxDepth"
      :max-length="maxLength"
      :currentDepth="0"
    />
  </div>
</template>

<script>
import _ from "lodash";
import TreeViewItem from "./TreeViewItem.vue";

export default {
  name: "TreeView",
  components: {
    TreeViewItem
  },
  props: ["data", "max-depth", "max-length", "root"],
  methods: {
    // Transformer for the non-Collection types,
    // like String, Integer of Float
    transformValue: function(valueToTransform, keyForValue) {
      return {
        key: keyForValue,
        type: "value",
        value: valueToTransform
      };
    },
    transformConstant: function(constToTransform) {
      return {
        type: "const",
        const: constToTransform
      };
    },
    transformLambda: function(lambdaToTransform, keyForLambda) {
      return {
        key: keyForLambda,
        type: "lambda",
        lambda: lambdaToTransform.args
      };
    },

    // Since we use lodash, the _.map method will work on
    // both Objects and Arrays, returning either the Key as
    // a string or the Index as an integer
    generateChildrenFromCollection: function(collection) {
      return _.map(collection, (value, keyOrIndex) => {
        if (this.isLambda(value)) {
          return this.transformLambda(value, keyOrIndex);
        }
        if (this.isTypedObject(value)) {
          return this.transformTypedObject(value, keyOrIndex);
        }
        if (this.isObject(value)) {
          return this.transformObject(value, keyOrIndex);
        }
        if (this.isArray(value)) {
          return this.transformArray(value, keyOrIndex);
        }
        if (this.isValue(value)) {
          return this.transformValue(value, keyOrIndex);
        }
      });
    },

    // Transformer for the Array type
    transformArray: function(arrayToTransform, keyForArray, isRootObject = false, klass = "") {
      return {
        key: keyForArray,
        type: "array",
        isRoot: isRootObject,
        klass: klass,
        children: this.generateChildrenFromCollection(arrayToTransform)
      };
    },

    transformTypedObject: function(objectToTransform, keyForObject) {
      let keys = Object.keys(objectToTransform);
      let comment = keys[0];
      let child = objectToTransform[keys[0]];

      if (keys[0] === "option" && this.isTypedObject(child)) {
        let child_keys = Object.keys(child);
        comment += " " + child_keys[0];
        child = child[child_keys[0]];
      }

      if (this.isValue(child)) {
        child = [child];
      }

      return {
        key: keyForObject,
        type: "object",
        isRoot: false,
        comment: comment,
        children: this.generateChildrenFromCollection(child)
      };
    },

    // Transformer for the Object type
    transformObject: function(objectToTransform, keyForObject, isRootObject = false, klass = "") {
      return {
        key: keyForObject,
        type: "object",
        isRoot: isRootObject,
        klass: klass,
        children: this.generateChildrenFromCollection(objectToTransform)
      };
    },

    // Helper Methods for value type detection
    isObject: function(value) {
      return _.isPlainObject(value);
    },

    isArray: function(value) {
      return _.isArray(value);
    },

    isValue: function(value) {
      return !this.isObject(value) && !this.isArray(value);
    },

    isTypedObject: function(value) {
      if (this.isObject(value)) {
        let keys = Object.keys(value);
        if (keys.length === 1) {
          return ["map", "big_map", "list", "set", "option", "lambda", "contract", "or"].includes(
            keys[0]
          );
        }
      }
      return false;
    },

    isLambda: function(value) {
      return this.isObject(value) && value.prim === "Lambda";
    }
  },
  computed: {
    parsedData: function() {
      if (this.isValue(this.data)) {
        return this.transformConstant(this.data);
      }
      if (this.isArray(this.data)) {
        let klass = "tree-view-wrapper-array";
        if (this.data.some(function(x) { return this.isArray(x) || this.isObject(x) }, this)) {
          klass += " tree-view-wrapper-chevron";
        }
        return this.transformArray(this.data, this.root, true, klass);
      }

      let keys = Object.keys(this.data);

      if (keys.length === 1) {
        let value = this.data[keys[0]];
        if (value === "Unit" || value === null) {
          return this.transformConstant(keys[0]);
        }
      }

      let klass = "tree-view-wrapper-object";
      if (Object.values(this.data).some(function(x) { return this.isArray(x) || this.isObject(x) }, this)) {
        klass += " tree-view-wrapper-chevron";
      }
      return this.transformObject(this.data, this.root, true, klass);
    }
  }
};
</script>

<style scoped>
/* DIRTY HACK BEGIN */

.tree-view-wrapper-array {
  margin-left: -18px;
  overflow: auto;
}

.tree-view-wrapper-object {
  margin-left: -18px;
  overflow: auto;
}

/* DIRTY HACK END */
</style>
