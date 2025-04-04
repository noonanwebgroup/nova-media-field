<template>
    <DefaultField
        :field="currentField"
        :errors="errors"
        :show-help-text="showHelpText"
        :full-width-content="fullWidthContent"
    >
        <template #field>
            <div class="nova-media-field">
                <div class="grid gap-4">
                    <DropZone
                        v-if="!currentField.readonly"
                        :attribute="currentField.attribute"
                        :multiple="currentField.multiple"
                        :disabled="currentField.readonly"
                        :accepted-types="currentField.acceptedTypes"
                        :dusk="`${currentField.attribute}-delete-link`"
                        :file-manager="currentField?.fileManager || false"
                        @fileChanged="handleFileChange"
                        @openFileManager="fileManagerState = true"
                    />

                    <Gallery
                        :value="currentField.value"
                        :field="currentField"
                        :multiple="currentField.multiple"
                        :readonly="currentField.readonly"
                        :errors="errors"
                        @input="currentField.value = $event"
                    />
                </div>
            </div>

            <Teleport to="body" v-if="currentField?.fileManager || false">
                <BrowserModal
                    :multiple="currentField.multiple"
                    :selecting="true"
                    v-model:state="fileManagerState"
                    @confirmSelection="confirmSelection"
                />
            </Teleport>
        </template>
    </DefaultField>
</template>

<script>
import { DependentFormField, HandlesValidationErrors } from "laravel-nova";
import uniqid from "uniqid";
import { serialize } from "object-to-formdata";
import DropZone from "../DropZone.vue";
import { BrowserModal } from "@vendor/stepanenko3/nova-filemanager/dist/js/package.js";

export default {
    mixins: [DependentFormField, HandlesValidationErrors],

    props: ["resourceName", "resourceId", "field"],

    components: {
        BrowserModal,
        DropZone,
    },

    data: () => ({
        originalValue: null,
        fileManagerState: false,
    }),

    mounted() {
        this.originalValue = this.field.value.slice();
    },

    methods: {
        confirmSelection(files) {
            const orderStart =
                this.currentField.value.length > 0
                    ? Math.max(...this.currentField.value.map((o) => o.order_column))
                    : 0;

            const filesForUpload = files.map(async (file) => {
                const filePayload = {
                    id: file.id,
                    file_name: file.name,
                    mime_type: file.mime,
                    size: file.size,
                    order_column: orderStart + 1,
                    created_at: new Date(file.lastModifiedAt).toISOString(),
                    updated_at: new Date(file.lastModifiedAt).toISOString(),
                    conversion: {
                        original: file.url,
                    },
                    generated_conversions: {},
                    custom_properties: {
                        source: 'media',
                        path: file.url,
                    },
                    markForUpload: true,
                    file: new File(
                        [await fetch(file.url).then((r) => r.blob())],
                        file.name,
                        {
                            type: file.mime,
                        }
                    ),
                };

                if (this.currentField.multiple) {
                    this.currentField.value.push(filePayload);
                } else {
                    this.currentField.value = [filePayload];
                }
            });

            this.handleFileChange(filesForUpload);
        },

        handleFileChange(newFiles) {
            const files = Array.from(newFiles);
            const orderStart =
                this.currentField.value.length > 0
                    ? Math.max(...this.currentField.value.map((o) => o.order_column))
                    : 0;

            files.forEach(async (file, index) => {
                const fileId = uniqid();

                const filePayload = {
                    id: fileId,
                    file_name: file.name,
                    mime_type: file.type,
                    size: file.size,
                    order_column: orderStart + 1,
                    created_at: new Date(file.lastModified).toISOString(),
                    updated_at: new Date(file.lastModified).toISOString(),
                    conversion: {
                        original: URL.createObjectURL(file),
                    },
                    custom_properties: file.custom_properties || {},
                    generated_conversions: {},
                    markForUpload: true,
                    file: file,
                };

                if (this.currentField.multiple) {
                    this.currentField.value.push(filePayload);
                } else {
                    this.currentField.value = [filePayload];
                }
            });
        },

        getCustomProperties(file) {
            return (this.currentField.customPropertiesFields || []).reduce(
                (properties, { attribute: property }) => {
                    const value = file?.custom_properties
                        ? file.custom_properties[property]
                        : null;

                    if (value !== null) {
                        properties[property] = value;
                    }

                    return properties;
                },
                {}
            );
        },

        fill(formData) {
            const data = {};

            this.currentField.value.forEach((file) => {
                const request = {
                    id: file.id,
                    order: file.order_column,
                    custom_properties: this.getCustomProperties(file),
                };

                if (file.markForUpload && !file.error) {
                    request.file = file.file;
                }

                data[request.id] = request;
            });

            this.originalValue.forEach((originalFile) => {
                const index = this.currentField.value.findIndex(
                    (file) =>
                        originalFile.id === file.id && !file?.markForUpload
                );

                if (index === -1) {
                    data[originalFile.id] = {
                        id: originalFile.id,
                        delete: true,
                    };
                }
            });

            console.log(data);

            serialize(
                {
                    [`${this.currentField.attribute}`]: data,
                },
                {},
                formData
            );
        },
    },
};
</script>
