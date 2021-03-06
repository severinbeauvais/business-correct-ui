<template>
  <v-card flat id="alteration-summary" v-if="isSummaryMode">

    <confirm-dialog
      ref="confirm"
      attach="#app"
    />

    <!-- Section Header -->
    <div class="summary-header px-4 mb-4">
      <v-row no-gutters>
        <v-col cols="9">
          <img  class="my-n1" src="@/assets/images/currency-usd-circle.svg">
          <label class="summary-title">Alteration Notice Changes ($100.00 Fee)</label>
        </v-col>

        <!-- Actions -->
        <v-col cols="3" class="mt-n2">
          <div class="actions mr-4">
            <v-btn
              text color="primary"
              id="btn-change-alteration"
              @click="setSummaryMode(false)"
            >
              <v-icon small>mdi-pencil</v-icon>
              <span>Change</span>
            </v-btn>
            <v-btn
              text color="primary"
              id="btn-delete-alteration"
              @click="restoreOriginalSnapshot()"
            >
              <v-icon small>mdi-delete</v-icon>
              <span>Remove</span>
            </v-btn>
          </div>
        </v-col>
      </v-row>
    </div>

    <!-- Business Name -->
    <template v-if="hasBusinessNameChanged">
      <div class="section-container business-name-summary">
        <v-row no-gutters>
          <v-col cols="3">
            <label><strong>Company Name</strong></label>
          </v-col>

          <v-col cols="8" class="mt-n1">
            <div class="company-name font-weight-bold text-uppercase">{{ companyName }}</div>
            <div class="company-name mt-2">{{ getNameRequest.nrNumber }}</div>
          </v-col>
        </v-row>
      </div>
    </template>

    <!-- Business Type -->
    <template v-if="hasBusinessTypeChanged">
      <v-divider class="mx-4" />
      <div class="section-container business-type-summary">
        <v-row no-gutters>
          <v-col cols="3">
            <label><strong>Business Type</strong></label>
          </v-col>

          <v-col cols="8">
            <span class="info-text">Changing from a {{ getCorpTypeDescription(originalEntityType) }}</span>
            &nbsp;
            <span class="info-text">to a {{getCorpTypeDescription(getEntityType)}}</span>

            <p class="subtitle mt-2 pt-2">Benefit Company Articles</p>
            <div class="confirmed-msg">
              <v-icon color="success" class="confirmed-icon">mdi-check</v-icon>
              <span class="info-text text-body-3 confirmed-icon ml-2">
                The company has completed a set Benefit Company Articles containing a benefit provision, and a copy
                of these articles has been added to the company's record book.
              </span>
            </div>
          </v-col>
        </v-row>
      </div>
    </template>

    <!-- Name Translation -->
    <template v-if="hasNameTranslationChange">
      <v-divider class="mx-4" />
      <div class="section-container name-translation-summary">
        <name-translation
          :nameTranslations="getNameTranslations"
          :isSummaryMode="true"
        />
      </div>
    </template>

    <!-- Share Structure -->
    <template v-if="hasShareStructureChanged">
      <v-divider class="mx-4" />
      <div class="section-container share-structure-summary">
        <v-row no-gutters>
          <v-col cols="3">
            <label><strong>Share Structure</strong></label>
          </v-col>
        </v-row>
        <share-structures class="mt-6" :is-edit-mode="false" />
      </div>
    </template>

    <!-- Pre-existing Company Provisions -->
    <template v-if="getProvisionsRemoved">
      <v-divider class="mx-4" />
      <div class="section-container name-translation-summary">
        <v-row no-gutters>
          <v-col cols="3">
            <label><strong>Pre-existing<br>Company Provisions</strong></label>
          </v-col>

          <v-col cols="8">
            <span class="info-text">
              The company has resolved that none of the Pre-existing Company Provisions are to apply to this company.
            </span>
          </v-col>
        </v-row>
      </div>
    </template>

    <!-- Resolution or Court Order Dates -->
    <template v-if="getNewResolutionDates && getNewResolutionDates.length > 0">
      <v-divider class="mx-4" />
      <div class="section-container resolution-court-order-dates-summary">
        <resolution-dates
          :added-dates="getNewResolutionDates"
          :previous-dates="getPreviousResolutionDates"
          :isEditMode="false"
        />
      </div>
    </template>

    <!-- Alteration Date and Time -->
    <div class="ma-6 pb-6">
      <v-container class="alteration-date-time" :class="{ 'invalid': alterationDateTimeInvalid }">
        <v-row no-gutters>
          <v-col cols="3" class="inner-col-1">
            <label><strong>Alteration Date<br>and Time</strong></label>
          </v-col>

          <v-col cols="9" class="inner-col-2">
            <p class="info-text">Select the date and time of alteration of your business. You may select a date and
              time up to 10 days in the future (note: there is an <strong>additional fee of $100.00</strong> to enter
              an alteration date and time in the future). Unless a business has special requirements, most businesses
              select an immediate Alteration Date and Time.
            </p>

            <effective-date-time
              :currentJsDate="getCurrentJsDate"
              :effectiveDateTime="getEffectiveDateTime"
              @dateTimeString="setEffectiveDateTimeString($event)"
              @isFutureEffective="setIsFutureEffective($event); emitHaveChanges()"
              @valid="setEffectiveDateValid($event)"
            />

            <v-card flat class="px-16 pb-8 mt-n12" v-if="isFutureEffective && isEffectiveDateTimeValid">
              The alteration for this business will be effective as of:<br>
              <strong>{{effectiveDateTimeString}}</strong>
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </div>

  </v-card>
</template>

<script lang="ts">
import { Component, Emit, Mixins, Prop } from 'vue-property-decorator'
import { Action, Getter } from 'vuex-class'
import { ConfirmDialog } from '@/components/dialogs'
import {
  ActionBindingIF,
  BusinessSnapshotIF,
  ConfirmDialogType,
  EffectiveDateTimeIF,
  NameRequestIF,
  ShareClassIF,
  ShareStructureIF,
  ValidFlagsIF,
  NameTranslationIF
} from '@/interfaces'
import { CommonMixin, DateMixin, EnumMixin, FilingTemplateMixin, LegalApiMixin } from '@/mixins'
import { CorpTypeCd } from '@/enums'
import { EffectiveDateTime } from '@/components/common'
import { ShareStructures } from '@/components/ShareStructure'
import { ResolutionDates } from '@/components/Articles'
import { NameTranslation } from '@/components/YourCompany/NameTranslations'

@Component({
  components: {
    ConfirmDialog,
    EffectiveDateTime,
    ResolutionDates,
    ShareStructures,
    NameTranslation
  }
})
export default class AlterationSummary extends Mixins(
  CommonMixin,
  DateMixin,
  EnumMixin,
  FilingTemplateMixin,
  LegalApiMixin
) {
  // Refs
  $refs!: {
    confirm: ConfirmDialogType
  }

  // Global getters
  @Getter getCurrentJsDate!: Date
  @Getter getApprovedName!: string
  @Getter getBusinessNumber!: string
  @Getter getEntityType!: CorpTypeCd
  @Getter isSummaryMode!: boolean
  @Getter getNameRequest!: NameRequestIF
  @Getter getEffectiveDateTime!: EffectiveDateTimeIF
  @Getter getShareClasses!: ShareClassIF[]
  @Getter getSnapshotShareStructure!: ShareStructureIF
  @Getter getOriginalSnapshot!: BusinessSnapshotIF
  @Getter getNewResolutionDates!: string[]
  @Getter getPreviousResolutionDates!: string[]
  @Getter getNameTranslations!: NameTranslationIF[]

  // Alteration flag getters
  @Getter getAlterationValidFlags!: ValidFlagsIF
  @Getter hasBusinessNameChanged!: boolean
  @Getter hasBusinessTypeChanged!: boolean

  // Global actions
  @Action setSummaryMode!: ActionBindingIF
  @Action setEffectiveDateTimeString!: ActionBindingIF
  @Action setIsFutureEffective!: ActionBindingIF
  @Action setEffectiveDateValid!: ActionBindingIF

  /** Prop to perform validation. */
  @Prop() readonly validate: boolean

  get isFutureEffective (): boolean {
    return this.getEffectiveDateTime.isFutureEffective
  }

  get isEffectiveDateTimeValid (): boolean {
    return this.getAlterationValidFlags.isValidEffectiveDate
  }

  get effectiveDateTimeString (): string {
    const date = new Date(this.getEffectiveDateTime.dateTimeString)
    return this.fullFormatDate(date)
  }

  /** The company name (from NR, or incorporation number). */
  get companyName (): string {
    if (this.getApprovedName) return this.getApprovedName

    return `${this.getBusinessNumber || '[Incorporation Number]'} B.C. Ltd.`
  }

  get originalEntityType (): string {
    return this.getOriginalSnapshot?.businessInfo?.legalType
  }

  /** True if invalid class should be set for Alteration Date-Time container. */
  get alterationDateTimeInvalid (): boolean {
    return (this.validate && !this.getAlterationValidFlags.isValidEffectiveDate)
  }

  /** Local getter, using a mixin method to detect changes to Share Structure. */
  get hasShareStructureChanged (): boolean {
    return !this.isSame(this.getShareClasses, this.getSnapshotShareStructure?.shareClasses, ['action'])
  }

  get hasNameTranslationChange (): boolean {
    return this.getNameTranslations.length > 0 &&
      this.getNameTranslations.filter(x => x.action).length > 0
  }

  /** Restore baseline data to original snapshot. */
  restoreOriginalSnapshot (): void {
    // open confirmation dialog and wait for response
    this.$refs.confirm.open(
      'Remove Alteration',
      'All changes to your company information will be removed.',
      {
        width: '45rem',
        persistent: true,
        yes: 'Remove Alteration',
        no: null,
        cancel: 'Cancel'
      }
    ).then(() => {
      // Restore original data
      this.parseBusinessSnapshot()
      this.setSummaryMode(false)
    }).catch(async () => {
      // if we get here, no was clicked
      // nothing to do
    })
  }

  @Emit('haveChanges')
  emitHaveChanges (): void {}
}
</script>

<style lang="scss" scoped>
@import '@/assets/styles/theme.scss';

.summary-header {
  display: flex;
  background-color: $BCgovBlue5O;
  padding: 1.25rem;
}

.confirmed-msg {
  display: flex;
  .confirmed-icon, .confirmed-note {
    display: block;
  }
}

.summary-title {
  padding-left: 0.5rem;
}

.company-name {
  font-size: 1.5rem;
}

.actions {
  position: absolute;
  right: 0;

  .undo-action{
    border-right: 1px solid $gray1;
  }

  .v-btn {
    min-width: 0.5rem;
  }
}

.alteration-date-time {
  padding: 2rem;
  background-color: $gray1;

  &.invalid {
    border-left: 4px solid $BCgovInputError;
    padding-left: calc(2rem - 4px);

    label {
      color: $BCgovInputError;
    }
  }
}

.inner-col-1 {
  // adjustment to make this inner container column the same width as the outer columns
  // ie, decrease width by 1/2 container margin + padding
  flex: 0 0 calc(25% - 1.5rem);
}

.inner-col-2 {
  // adjustment to make this inner container column the same width as the outer columns
  // ie, increase width by 1/2 container margin + padding
  flex: 0 0 calc(75% + 1.5rem);
  max-width: calc(75% + 1.5rem);
}
</style>
