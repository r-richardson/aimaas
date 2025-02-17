<template>
  <Spinner :loading="loading">
    <div v-if="!loading && editEntity!== null" class="container">
      <TextInput label="Name" v-model="editEntity.name" :args="{ id: 'name', maxlength: 128 }"
                 :required="true"/>
      <TextInput label="Slug" v-model="editEntity.slug" :args="{ id: 'slug', maxlength: 128 }"
                 :required="true"/>

      <h3 class="mt-3">Attributes</h3>
      <template v-for="attr in schema?.attributes || []" :key="attr.name">
        <!-- ATTRIBUTE IS REFERENCE -->
        <template v-if="attr.type === 'FK'">
          <ReferencedEntitySelect v-model="editEntity[attr.name]" :label="attr.name"
                                  :args="{id: attr.name}" :fk-schema-id="attr.bound_schema_id"
                                  :select-type="attr.list ? 'many': 'single'"
                                  :required="requiredAttrs.includes(attr.name)"/>
        </template>
        <!-- ATTRIBUTE IS SINGLE VALUED -->
        <template v-else-if="!attr.list">
          <component :is="TYPE_INPUT_MAP[attr.type]" v-model="editEntity[attr.name]"
                     :label="attr.name" :args="{id: attr.name}"
                     :required="requiredAttrs.includes(attr.name)"/>
        </template>
        <!-- ATTRIBUTE IS MULTI VALUED -->
        <template v-else>
          <div class="row mb-1 pb-1 border-bottom border-light">
            <div class="col-lg-2">
              {{ attr.name }}
            </div>
            <div class="col-lg-10">
              <ul class="list-group">
                <li v-for="(val, idx) in (editEntity[attr.name] || [])"
                    :key="`${entity.id}-${attr.name}-${idx}`"
                    class="list-group-item d-flex">
                  <div class="flex-grow-1">
                    <component :is="TYPE_INPUT_MAP[attr.type]" v-model="editEntity[attr.name][idx]"
                               :vertical="true" :args="{id: `${attr.name}-${idx}`}"
                               :required="requiredAttrs.includes(attr.name)"/>
                  </div>
                  <div>
                    <button type="button" class="btn btn-outline-cta ms-1"
                            data-ts-toggle="tooltip" title="Remove"
                            @click="removeListItem(attr.name, idx)">
                      <i class="eos-icons">backspace</i>
                    </button>
                  </div>
                </li>
              </ul>
              <div class="d-grid gap-2">
                <button type="button" class="btn btn-outline-primary float-end mt-1"
                        data-ts-toggle="tooltip" title="Add value" @click="addListItem(attr.name)">
                  <i class="eos-icons">add</i>
                  Add
                </button>
              </div>
            </div>
          </div>
        </template>
      </template>
      <div class="d-grid gap-2 mt-1">
        <button type="button" class="btn btn-primary p-2" @click="saveChanges">
          <i class="eos-icons me-1">save</i>
          Save {{ batchMode? 'all changes' : 'changes'}}
        </button>
      </div>
    </div>
  </Spinner>
</template>

<script>
import _cloneDeep from "lodash/cloneDeep";
import _isEqual from "lodash/isEqual";

import TextInput from "@/components/inputs/TextInput";
import Checkbox from "@/components/inputs/Checkbox";
import DateTime from "@/components/inputs/DateTime";
import DateInput from "@/components/inputs/DateInput";
import IntegerInput from "@/components/inputs/IntegerInput";
import FloatInput from "@/components/inputs/FloatInput";
import ReferencedEntitySelect from "@/components/inputs/ReferencedEntitySelect";
import Spinner from "@/components/layout/Spinner";
import {TYPE_INPUT_MAP} from "@/utils";


export default {
  name: "EntityForm",
  components: {
    Spinner, TextInput, Checkbox, DateTime, DateInput, IntegerInput, FloatInput,
    ReferencedEntitySelect
  },
  props: {
    schema: {
      type: Object,
      required: true
    },
    entity: {
      type: Object,
      required: true
    },
    batchMode: {
      type: Boolean,
      default: false
    }
  },
  inject: ["updatePendingRequests"],
  emits: ["update", "save-all"],
  activated() {
    this.updateSchemaMeta();
  },
  watch: {
    schema: {
      handler: "updateSchemaMeta",
      deep: true,
      immediate: true
    },
    entity: {
      handler: "updateEntityData",
      deep: true,
      immediate: true
    }
  },
  data() {
    return {
      editEntity: null,
      loading: true,
      TYPE_INPUT_MAP
    };
  },
  computed: {
    requiredAttrs() {
      return (this.schema?.attributes || []).filter(a => a.required).map(a => a.name);
    }
  },
  methods: {
    addListItem(attrName) {
      if (this.editEntity[attrName] instanceof Array) {
        this.editEntity[attrName].push('');
      } else {
        console.error("Attribute", attrName, "is not a multi-value attribute!");
      }
    },
    removeListItem(attrName, idx) {
      try {
        this.editEntity[attrName].splice(idx, 1);
      } catch (e) {
        console.error("Failed to remove item from attribute list ", attrName);
      }
    },
    prepEntity() {
      const listAttrs = (this.schema?.attributes || []).filter(x => x.list);
      if (!this.entity) {
        this.editEntity = {name: null, slug: null};
        for (const attr of this.schema?.attributes || []) {
          this.editEntity[attr.name] = null;
        }
      }
      for (const attr of listAttrs) {
        if (this.editEntity && !this.editEntity[attr.name]) {
          this.editEntity[attr.name] = [];
        }
      }
    },
    async updateSchemaMeta() {
      this.loading = true;
       if (!this.entity) {
        this.prepEntity();
      }
      this.loading = false;
    },
    async updateEntityData() {
      if (this.entity?.schema_id && this.entity?.schema_id !== this.schema.id) {
        await this.updateSchemaMeta();
      }
      if (this.entity && this.editEntity?.id !== this.entity?.id) {
        this.editEntity = _cloneDeep(this.entity);
      } else if (!this.entity) {
        this.prepEntity();
      }
    },
    getChanges() {
      const result = {};
      for (const [attr, value] of Object.entries(this.editEntity)) {
        if (!_isEqual(value, this.entity[attr])) {
          result[attr] = value;
        }
      }
      return result;
    },
    async saveChanges() {
      if (this.batchMode) {
        this.$emit("save-all");
        return
      }

      return this.saveEntity();
    },
    async updateEntity() {
      this.loading = true;
      const body = this.getChanges();
      if (Object.keys(body).length === 0) {
        console.info("No changes, therefore pushing nothing to server.");
        this.loading = false;
        return null;
      }
      const response = await this.$api.updateEntity({
        schemaSlug: this.schema.slug,
        entityIdOrSlug: this.entity.id,
        body: body,
      });
      this.loading = false;
      return response
    },
    async createEntity() {
      this.loading = true;
      const response = await this.$api.createEntity({
        schemaSlug: this.$route.params.schemaSlug,
        body: this.editEntity
      });
      this.loading = false;
      return response;
    },
    async saveEntity() {
      this.loading = true;
      let response;
      if (this.$route.params.entitySlug) {
        // UPDATE EXISTING ENTITY
        response = this.updateEntity();
      } else {
        // CREATE NEW ENTITY
        response = this.createEntity();
      }
      if (response) {
        this.updatePendingRequests();
      }
    },
  },
}
</script>

<style scoped>

</style>
