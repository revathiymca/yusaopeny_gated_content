<template>
  <div class="gated-content-video-page virtual-meeting-page">
    <div v-if="loading" class="text-center">
      <Spinner></Spinner>
    </div>
    <div v-else-if="error">Error loading</div>
    <template v-else>
      <div
        class="virtual-meeting-page__image"
        :style="coverStyle"
      >
        <div class="virtual-meeting-page__link">
          <a :href="meetingLink.uri" target="_blank" class="btn btn-lg btn-primary">
            {{ meetingLink.title }}
          </a>
        </div>
      </div>
      <div class="video-footer-wrapper bg-white">
        <div class="video-footer gated-containerV2 py-40-20 px--20-10 text-black">
          <div class="pb-20-10 cachet-book-32-28">{{ video.attributes.title }}</div>
          <div class="video-footer__fav pb-40-20">
            <AddToFavorite
              :id="video.attributes.drupal_internal__id"
              :type="'eventinstance'"
              :bundle="'virtual_meeting'"
              class="rounded-border border-concrete"
            ></AddToFavorite>
            <AddToCalendar :event="event"></AddToCalendar>
            <div class="timer" :class="{live: isOnAir}">
              <template v-if="isOnAir">
                LIVE!
              </template>
              <template v-else>
                Starts in {{ startsIn }}
              </template>
            </div>
          </div>
          <div class="verdana-14-12 text-thunder">
            <div class="video-footer__block">
              <SvgIcon icon="date-icon" class="fill-gray" :growByHeight=false></SvgIcon>
              {{ date }}
            </div>
            <div class="video-footer__block">
              <SvgIcon icon="clock-regular" class="fill-gray" :growByHeight=false></SvgIcon>
              {{ time }} ({{ duration }})
            </div>
            <div class="video-footer__block" v-if="instructors && instructors.length > 0">
              <SvgIcon icon="instructor-icon" class="fill-gray" :growByHeight=false></SvgIcon>
              <ul>
                <li v-for="instructor in instructors" :key="instructor.drupal_internal__tid">
                  <router-link :to="{ name: 'Instructor', params: { id: instructor.uuid }}">
                    {{ instructor.name }}
                  </router-link>
                </li>
              </ul>
            </div>
            <div
              class="video-footer__block"
              v-if="level"
            >
              <SvgIcon icon="difficulty-icon-grey" :css-fill="false"></SvgIcon>
              {{ level | capitalize }}
            </div>
            <div class="video-footer__block video-footer__category"
                 v-if="category && category.length > 0">
              <SvgIcon icon="categories" class="fill-gray" :growByHeight=false></SvgIcon>
              <ul>
                <li
                  v-for="tid in category.map(item => item.drupal_internal__tid)"
                  class="video-footer__category-list-item"
                  :key="tid"
                >
                  <CategoryLinks :tid="tid" />
                </li>
              </ul>
            </div>
            <div
              v-if="video.attributes.equipment.length > 0"
              class="video-footer__block">
              <SvgIcon icon="cubes-solid" :growByHeight=false></SvgIcon>
              Equipment:
            </div>
            <ul class="video-footer__equipment">
              <li v-for="equip in video.attributes.equipment"
                  :key="equip.drupal_internal__tid">
                {{ equip.name }}
              </li>
            </ul>
          </div>
          <div
            v-if="description"
            class="verdana-16-14"
            v-html="descriptionProcessed"
          ></div>
        </div>
      </div>
      <EventListing
        :title="config.components.virtual_meeting.up_next_title"
        :excluded-video-id="video.id"
        :eventType="'virtual_meeting'"
        :viewAll="true"
        :limit="8"
      />
    </template>
  </div>
</template>

<script>
import client from '@/client';
import AddToFavorite from '@/components/AddToFavorite.vue';
import Spinner from '@/components/Spinner.vue';
import EventListing from '@/components/event/EventListing.vue';
import AddToCalendar from '@/components/event/AddToCalendar.vue';
import CategoryLinks from '@/components/category/CategoryLinks.vue';
import { JsonApiCombineMixin } from '@/mixins/JsonApiCombineMixin';
import { EventMixin } from '@/mixins/EventMixin';
import { SeriesEventMixin } from '@/mixins/SeriesEventMixin';
import { ImageStyleMixin } from '@/mixins/ImageStyleMixin';
import SvgIcon from '@/components/SvgIcon.vue';

export default {
  name: 'VirtualMeetingPage',
  mixins: [JsonApiCombineMixin, EventMixin, SeriesEventMixin, ImageStyleMixin],
  components: {
    SvgIcon,
    AddToFavorite,
    EventListing,
    AddToCalendar,
    Spinner,
    CategoryLinks,
  },
  props: {
    id: {
      type: String,
      required: true,
    },
  },
  data() {
    return {
      loading: true,
      error: false,
      video: null,
      response: null,
      params: [
        'field_ls_category',
        'field_ls_level',
        'field_ls_image',
        'field_ls_image.field_media_image',
        'field_gc_instructor_reference',
        // Data from parent (series).
        'category',
        'level',
        'equipment',
        'image',
        'image.field_media_image',
        'instructor_reference',
      ],
    };
  },
  computed: {
    // This values most of all from parent (series), but can be overridden by item,
    // so ve need to check this here and use correct value.
    image() {
      return this.video.attributes['field_ls_image.field_media_image']
        ?? this.video.attributes['image.field_media_image'];
    },
    coverStyle() {
      if (!this.image) { return null; }
      return { backgroundImage: `url(${this.getStyledUrl(this.image, 'carnation_banner_1920_700')})` };
    },
    meetingLink() {
      const link = {
        title: 'Join Meeting',
      };
      if (this.video.attributes.field_vm_link) {
        link.uri = this.video.attributes.field_vm_link.uri;
        if (this.video.attributes.field_vm_link.title) {
          link.title = this.video.attributes.field_vm_link.title;
        }
      }
      if (this.video.attributes.meeting_link) {
        link.uri = this.video.attributes.meeting_link.uri;
        if (this.video.attributes.meeting_link.title) {
          link.title = this.video.attributes.meeting_link.title;
        }
      }
      return link;
    },
    descriptionProcessed() {
      // Handle D9 description fields.
      if (typeof this.description === 'string') {
        return this.description ? this.description.processed : '';
      }
      // Handle D10 description fields.
      if (typeof this.description === 'object') {
        return this.description ? this.description[0].processed : '';
      }
      return '';
    },
    event() {
      return {
        start: this.formatDate(this.video.attributes.date.value),
        duration: [this.getDuration(this.video.attributes.date), 'hour'],
        title: this.video.attributes.title,
        description: `${this.meetingLink.title}: ${this.meetingLink.uri} <br> Virtual meeting page: ${this.pageUrl}`,
        busy: true,
        guests: [],
      };
    },
  },
  methods: {
    async load() {
      this.loading = true;
      const params = {};
      if (this.params) {
        params.include = this.params.join(',');
      }
      client
        .get(`jsonapi/eventinstance/virtual_meeting/${this.id}`, { params })
        .then((response) => {
          this.video = this.combine(response.data.data, response.data.included, this.params);
          this.multipleReferencesWorkaround(response);
          this.loading = false;
        }).then(() => {
          this.$log.trackEvent('entityView', 'series', 'virtual_meeting', this.video.attributes.drupal_internal__id);
        })
        .catch((error) => {
          this.error = true;
          this.loading = false;
          console.error(error);
          throw error;
        });
    },
  },
};
</script>
