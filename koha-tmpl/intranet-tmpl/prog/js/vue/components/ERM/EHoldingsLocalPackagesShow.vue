<template>
    <div v-if="!initialized">{{ $__("Loading") }}</div>
    <div v-else-if="erm_package" id="packages_show">
        <h2>
            {{ $__("Package #%s").format(erm_package.package_id) }}
            <span class="action_links">
                <router-link
                    :to="`/cgi-bin/koha/erm/eholdings/local/packages/edit/${erm_package.package_id}`"
                    :title="$__('Edit')"
                    ><i class="fa fa-pencil"></i
                ></router-link>

                <router-link
                    :to="`/cgi-bin/koha/erm/eholdings/local/packages/delete/${erm_package.package_id}`"
                    :title="$__('Delete')"
                    ><i class="fa fa-trash"></i
                ></router-link>
            </span>
        </h2>
        <div>
            <fieldset class="rows">
                <ol>
                    <li>
                        <label>{{ $__("Package name") }}:</label>
                        <span>
                            {{ erm_package.name }}
                        </span>
                    </li>
                    <li v-if="erm_package.vendor">
                        <label>{{ $__("Vendor") }}:</label>
                        <span>
                            <a
                                :href="`/cgi-bin/koha/acqui/booksellers.pl?booksellerid=${erm_package.vendor_id}`"
                                >{{ erm_package.vendor.name }}</a
                            >
                        </span>
                    </li>
                    <li>
                        <label>{{ $__("Package type") }}:</label>
                        <span>{{
                            get_lib_from_av(
                                "av_package_types",
                                erm_package.package_type
                            )
                        }}</span>
                    </li>
                    <li>
                        <label>{{ $__("Content type") }}:</label>
                        <span>{{
                            get_lib_from_av(
                                "av_package_content_types",
                                erm_package.content_type
                            )
                        }}</span>
                    </li>
                    <li>
                        <label>{{ $__("Notes") }}:</label>
                        <span>{{ erm_package.notes }}</span>
                    </li>
                    <li v-if="erm_package.created_on">
                        <label>{{ $__("Created on") }}:</label>
                        <span>{{ format_date(erm_package.created_on) }}</span>
                    </li>
                    <li v-if="erm_package.package_agreements.length">
                        <label>{{ $__("Agreements") }}</label>
                        <div
                            v-for="package_agreement in erm_package.package_agreements"
                            :key="package_agreement.agreement_id"
                        >
                            <router-link
                                :to="`/cgi-bin/koha/erm/agreements/${package_agreement.agreement.agreement_id}`"
                                >{{
                                    package_agreement.agreement.name
                                }}</router-link
                            >
                        </div>
                    </li>
                    <li>
                        <label>{{
                            $__("Titles (%s)").format(
                                erm_package.resources_count
                            )
                        }}</label>
                        <div v-if="erm_package.resources_count">
                            <EHoldingsPackageTitlesList
                                :package_id="erm_package.package_id.toString()"
                            />
                        </div>
                    </li>

                    <li></li>
                </ol>
            </fieldset>
            <fieldset class="action">
                <router-link
                    to="/cgi-bin/koha/erm/eholdings/local/packages"
                    role="button"
                    class="cancel"
                    >{{ $__("Close") }}</router-link
                >
            </fieldset>
        </div>
    </div>
</template>

<script>
import { inject } from "vue"
import EHoldingsPackageTitlesList from "./EHoldingsLocalPackageTitlesList.vue"
import { fetchLocalPackage } from "../../fetch"

export default {
    setup() {
        const format_date = $date

        const AVStore = inject("AVStore")
        const { get_lib_from_av } = AVStore

        return {
            format_date,
            get_lib_from_av,
        }
    },
    data() {
        return {
            erm_package: {
                package_id: null,
                vendor_id: null,
                name: "",
                external_id: "",
                package_type: "",
                content_type: "",
                created_on: null,
                resources: null,
                package_agreements: [],
            },
            initialized: false,
        }
    },
    beforeRouteEnter(to, from, next) {
        next(vm => {
            vm.getPackage(to.params.package_id)
        })
    },
    beforeRouteUpdate(to, from) {
        this.erm_package = this.getPackage(to.params.package_id)
    },
    methods: {
        async getPackage(package_id) {
            const erm_package = await fetchLocalPackage(package_id)
            this.erm_package = erm_package
            this.initialized = true
        },
    },
    components: {
        EHoldingsPackageTitlesList,
    },
    name: "EHoldingsLocalPackagesShow",
}
</script>
<style scoped>
.action_links a {
    padding-left: 0.2em;
    font-size: 11px;
}
fieldset.rows label {
    width: 25rem;
}
</style>
