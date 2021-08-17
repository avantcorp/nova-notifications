<template>
  <dropdown class="ml-auto h-9 flex items-center dropdown-right">
    <dropdown-trigger class="h-9 flex items-center">
        <span class="relative">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 text-90" fill="none" viewBox="0 0 24 24"
                 stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                    d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9"/>
            </svg>
            <span
                class="absolute -mt-4 -mr-1 text-xs bg-danger text-danger-light text-sm font-bold px-1 shadow-lg rounded-lg"
                style="right: 0;"
                v-if="count > 0">
                <span v-if="count < 9">{{ count }}</span>
                <span v-else>9+</span>
            </span>
        </span>
    </dropdown-trigger>

    <dropdown-menu slot="menu" width="600" direction="rtl">
      <loading-view :loading="isLoading">
        <div class="flex justify-between bg-40 text-90 p-4">
          <h3>{{ __('Notifications') }}</h3>

          <button v-if="count !== 0" class="btn btn-outline btn-white btn-xs font-normal" @click="markAllAsRead()">
            {{ __('Mark all as read') }}
          </button>
        </div>
        <p v-if="count === 0" class="block p-4 text-sm font-normal">
          {{ __('No new notifications') }}
        </p>
        <scroll-wrap v-else height="350">
          <slot>
            <notification-item
                :ref="'notification-' + notification.id"
                v-for="notification in notifications"
                :key="notification.id"
                :notification="notification"
            >
            </notification-item>
          </slot>
        </scroll-wrap>
      </loading-view>
    </dropdown-menu>
  </dropdown>
</template>

<script>
export default {
  data() {
    return {
      count: 0,
      notifications: [],
      isLoading: true,
    }
  },
  mounted() {
    const self = this

    self.loadNotifications(function (response) {
      Echo.private(Nova.config.user_model_namespace + '.' + response.data.user_id)
          .notification(self.notificationReceived)
    })

    Nova.$on('notification-read', function (e) {
      self.notifications[e.notification.id] = e.notification
      self.count--;
    })
  },
  methods: {
    loadNotifications: function (callback) {
      axios
          .get('/nova-vendor/nova-notifications/unread')
          .then(response => {
            this.isLoading = false
            this.count = response.data.count
            this.notifications = response.data.notifications

            if (callback) {
              callback(response)
            }
          })
    },
    notificationReceived: function (notification) {
      const self = this

      self.loadNotifications()

      let level = 'info'
      const levels = ['success', 'info', 'error']

      if (levels.indexOf(notification.level) !== -1) {
        level = notification.level
      }

      const markAsRead = {
        text: self.__('Mark as Read'),
        onClick: (e, toast) => {
          self.$refs['notification-' + notification.id][0].markAsRead()
          toast.goAway(0);
        }
      }

      const cancel = {
        text: self.__('Cancel'),
        onClick: (e, toast) => {
          toast.goAway(0);
        }
      }

      let actions = []

      if (notification.show_mark_as_read) {
        actions.push(markAsRead)
      }

      if (notification.show_cancel) {
        actions.push(cancel)
      }

      self.$toasted.show(notification.title, {
        type: level,
        keepOnHover: true,
        icon: notification.icon || null,
        iconPack: 'custom-class',
        action: actions,
      })
    },
    markAllAsRead: function () {
      axios
          .patch('/nova-vendor/nova-notifications')
          .then(() => {
            this.notifications = []
            this.count = 0
          })
    }
  }
}
</script>

<style scoped>
#max-height {
  max-height: 30rem;
}

.bg-light-gray {
  background-color: #EEEEEE;
}

.bg-light-gray:hover {
  background-color: #fefefe;
}
</style>
