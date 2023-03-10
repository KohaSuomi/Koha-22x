<template>
    <div>
        <fieldset>
            {{ $__("Publication title") }}:
            <input
                type="text"
                id="publication_title_filter"
                v-model="filters.publication_title"
                @keyup.enter="filter_table"
            />
            {{ $__("Publication type") }}:
            <select
                id="publication_type_filter"
                v-model="filters.publication_type"
            >
                <option value="">{{ $__("All") }}</option>
                <option
                    v-for="type in av_title_publication_types"
                    :key="type.authorised_values"
                    :value="type.authorised_value"
                >
                    {{ type.lib }}
                </option>
            </select>
            {{ $__("Selection status") }}:
            <select id="selection_type_filter" v-model="filters.selection_type">
                <option value="0">{{ $__("All") }}</option>
                <option value="1">{{ $__("Selected") }}</option>
                <option value="2">{{ $__("Not selected") }}</option>
            </select>
            <input
                @click="filter_table"
                id="filter_table"
                type="button"
                :value="$__('Submit')"
            />
            <span v-if="cannot_search">{{
                $__("Please enter a search term")
            }}</span>
        </fieldset>

        <!-- We need to display the table element to initiate DataTable -->
        <div
            id="title_list_result"
            :style="show_table ? 'display: block' : 'display: none'"
        >
            <div
                v-if="
                    local_title_count !== undefined &&
                    local_title_count !== null
                "
            >
                <router-link :to="local_titles_url">
                    {{
                        $__("%s titles found locally").format(local_title_count)
                    }}</router-link
                >
            </div>
            <div id="title_list_result" class="page-section">
                <table :id="table_id"></table>
            </div>
        </div>
    </div>
</template>

<script>
import { inject, createVNode, render } from "vue"
import { storeToRefs } from "pinia"
import { fetchLocalTitleCount } from "./../../fetch"
import {
    useDataTable,
    build_url_params,
    build_url,
} from "../../composables/datatables"

export default {
    setup() {
        const vendorStore = inject("vendorStore")
        const { vendors } = storeToRefs(vendorStore)

        const AVStore = inject("AVStore")
        const { av_title_publication_types } = storeToRefs(AVStore)
        const { get_lib_from_av } = AVStore

        const table_id = "title_list"
        useDataTable(table_id)

        return {
            vendors,
            av_title_publication_types,
            get_lib_from_av,
            erm_providers,
            table_id,
        }
    },
    data: function () {
        return {
            titles: [],
            initialized: true,
            filters: {
                publication_title: this.$route.query.publication_title || "",
                publication_type: this.$route.query.publication_type || "",
                selection_type: this.$route.query.selection_type || "",
            },
            cannot_search: false,
            show_table: false,
            local_title_count: null,
        }
    },
    computed: {
        local_titles_url() {
            return build_url(
                "/cgi-bin/koha/erm/eholdings/local/titles",
                this.filters
            )
        },
    },
    beforeRouteEnter(to, from, next) {
        next(vm => {
            vm.build_datatable()
        })
    },
    methods: {
        show_title: function (title_id) {
            this.$router.push(
                "/cgi-bin/koha/erm/eholdings/ebsco/titles/" + title_id
            )
        },
        filter_table: async function () {
            if (this.filters.publication_title.length) {
                this.cannot_search = false
                let new_route = build_url(
                    "/cgi-bin/koha/erm/eholdings/ebsco/titles",
                    this.filters
                )
                this.$router.push(new_route)
                this.show_table = true
                this.local_title_count = null
                $("#" + this.table_id)
                    .DataTable()
                    .draw()
                if (this.erm_providers.includes("local")) {
                    this.local_title_count = await fetchLocalTitleCount(
                        this.filters
                    )
                }
            } else {
                this.cannot_search = true
            }
        },
        build_datatable: function () {
            let show_title = this.show_title
            let get_lib_from_av = this.get_lib_from_av
            if (!this.show_table) {
                this.show_table = build_url_params(this.filters).length
                    ? true
                    : false
            }
            let filters = this.filters
            let table_id = this.table_id

            window["vendors"] = this.vendors.map(e => {
                e["_id"] = e["id"]
                e["_str"] = e["name"]
                return e
            })
            let vendors_map = this.vendors.reduce((map, e) => {
                map[e.id] = e
                return map
            }, {})
            window["av_title_publication_types"] =
                this.av_title_publication_types.map(e => {
                    e["_id"] = e["authorised_value"]
                    e["_str"] = e["lib"]
                    return e
                })

            let additional_filters = {
                publication_title: function () {
                    return filters.publication_title || ""
                },
                publication_type: function () {
                    return filters.publication_type || ""
                },
                selection_type: function () {
                    return filters.selection_type || ""
                },
            }
            $("#" + table_id).kohaTable(
                {
                    ajax: {
                        url: "/api/v1/erm/eholdings/ebsco/titles",
                    },
                    ordering: false,
                    dom: '<"top pager"<"table_entries"ilp>>tr<"bottom pager"ip>',
                    aLengthMenu: [
                        [10, 20, 50, 100],
                        [10, 20, 50, 100],
                    ],
                    deferLoading: true,
                    autoWidth: false,
                    columns: [
                        {
                            title: __("Title"),
                            data: "me.publication_title",
                            searchable: false,
                            orderable: false,
                            render: function (data, type, row, meta) {
                                // Rendering done in drawCallback
                                return ""
                            },
                        },
                        {
                            title: __("Publisher name"),
                            data: "me.publisher_name",
                            searchable: false,
                            orderable: false,
                            render: function (data, type, row, meta) {
                                return escape_str(row.publisher_name)
                            },
                        },
                        {
                            title: __("Publication type"),
                            data: "publication_type",
                            searchable: false,
                            orderable: false,
                            render: function (data, type, row, meta) {
                                return escape_str(
                                    get_lib_from_av(
                                        "av_title_publication_types",
                                        row.publication_type
                                    )
                                )
                            },
                        },
                        {
                            title: __("Identifier"),
                            data: "print_identifier:online_identifier",
                            searchable: false,
                            orderable: false,
                            render: function (data, type, row, meta) {
                                let print_identifier = row.print_identifier
                                let online_identifier = row.online_identifier
                                return (
                                    (print_identifier
                                        ? escape_str(
                                              _("ISBN (Print): %s").format(
                                                  print_identifier
                                              )
                                          )
                                        : "") +
                                    (online_identifier
                                        ? escape_str(
                                              _("ISBN (Online): %s").format(
                                                  online_identifier
                                              )
                                          )
                                        : "")
                                )
                            },
                        },
                    ],
                    drawCallback: function (settings) {
                        var api = new $.fn.dataTable.Api(settings)

                        if (!api.rows({ search: "applied" }).count()) return

                        $.each(
                            $(this).find("tbody tr td:first-child"),
                            function (index, e) {
                                let tr = $(this).parent()
                                let row = api.row(tr).data()
                                if (!row) return // Happen if the table is empty
                                let n = createVNode(
                                    "a",
                                    {
                                        role: "button",
                                        onClick: e => {
                                            e.preventDefault()
                                            show_title(row.title_id)
                                        },
                                    },
                                    `${row.publication_title} (#${row.title_id})`
                                )

                                if (row.is_selected) {
                                    n = createVNode("span", {}, [
                                        n,
                                        " ",
                                        createVNode("i", {
                                            class: "fa fa-check-square-o",
                                            style: {
                                                color: "green",
                                                float: "right",
                                            },
                                            title: __("Is selected"),
                                        }),
                                    ])
                                }
                                render(n, e)
                            }
                        )
                    },
                },
                null,
                0,
                additional_filters
            )

            if (filters.publication_title.length) {
                this.filter_table()
            }
        },
    },
    name: "EHoldingsEBSCOTitlesList",
}
</script>
