<<<<<<< HEAD
<template>
    <div class="kiwi-messagelist" @click="onThreadClick" @scroll.self="onThreadScroll">
        <div
            v-if="shouldShowChathistoryTools"
            class="kiwi-messagelist-scrollback"
        >
            <a @click="buffer.requestScrollback()" class="u-link">{{$t('messages_load')}}</a>
        </div>

        <div
            v-for="(message, idx) in filteredMessages"
            key="message.time"
            class="kiwi-messagelist-message"
            v-bind:class="[
                filteredMessages[idx-1] &&
                filteredMessages[idx-1].nick === message.nick &&
                message.time - filteredMessages[idx-1].time < 60000 &&
                message.type !== 'traffic' ?
                    'kiwi-messagelist-message-repeat' :
                    '',
                'kiwi-messagelist-message-' + message.type,
                message.type_extra ?
                    'kiwi-messagelist-message-' + message.type + '-' + message.type_extra :
                    '',
                isMessageHighlight(message) ?
                    'kiwi-messagelist-message--highlight' :
                    '',
                message.nick && message.nick.toLowerCase() === hover_nick.toLowerCase() ?
                    'kiwi-messagelist-message--hover' :
                    '',
                buffer.last_read && message.time > buffer.last_read ?
                    'kiwi-messagelist-message--unread' :
                    '',
                message.nick.toLowerCase() === ourNick.toLowerCase() ?
                    'kiwi-messagelist-message--own' :
                    '',
            ]"
        >
            <div
                v-if="bufferSetting('show_timestamps')"
                class="kiwi-messagelist-time"
            >
                {{formatTime(message.time)}}
            </div>
            <div
                class="kiwi-messagelist-nick"
                v-bind:style="nickStyle(message.nick)"
                v-bind:data-nick="message.nick"
                @mouseover="hover_nick=message.nick.toLowerCase();"
                @mouseout="hover_nick='';"
            >{{message.nick}}</div>
            <div class="kiwi-messagelist-body" v-html="formatMessage(message.message)"></div>
        </div>

        <not-connected
            v-if="buffer.getNetwork().state !== 'connected'"
            :buffer="buffer"
            :network="buffer.getNetwork()"
        ></not-connected>
    </div>
</template>

<script>

import strftime from 'strftime';
import _ from 'lodash';
import state from 'src/libs/state';
import * as TextFormatting from 'src/helpers/TextFormatting';
import NotConnected from './NotConnected';

export default {
    components: {
        NotConnected,
    },
    data: function data() {
        return {
            auto_scroll: true,
            chathistoryAvailable: true,
            hover_nick: '',
        };
    },
    props: ['buffer', 'messages', 'users'],
    computed: {
        useExtraFormatting: function useExtraFormatting() {
            // Enables simple markdown formatting
            return this.buffer.setting('extra_formatting');
        },
        filteredMessages: function filteredMessages() {
            let network = this.buffer.getNetwork();
            let currentNick = network.nick;

            let messages = this.messages.slice(0, this.messages.length);
            messages.sort((a, b) => {
                if (a.time > b.time) {
                    return 1;
                } else if (b.time > a.time) {
                    return -1;
                }

                return 0;
            });

            let list = [];
            let maxSize = this.buffer.setting('scrollback_size');
            let showJoinParts = this.buffer.setting('show_joinparts');
            for (let i = messages.length - 1; i >= 0 && list.length < maxSize; i--) {
                if (!showJoinParts && messages[i].type === 'traffic') {
                    continue;
                }
                // Ignored users have the ignore flag set
                if (messages[i].ignore) {
                    continue;
                }

                // When we join a channel the topic is usually sent next. But this looks
                // ugly when rendered. So we switch the topic + join messages around so
                // that the topic is first in the message list.
                if (
                    messages[i].type === 'topic' &&
                    messages[i - 1] &&
                    messages[i - 1].type === 'traffic' &&
                    messages[i - 1].nick === currentNick
                ) {
                    list.push(messages[i - 1]);
                    list.push(messages[i]);
                    i--;
                } else {
                    list.push(messages[i]);
                }
            }

            return list.reverse();
        },
        usersAsObject: function usersAsObject() {
            return this.buffer.users.reduce((prev, cur) => {
                prev[cur] = true;
                return prev;
            }, {});
        },
        shouldShowChathistoryTools: function shouldShowChathistoryTools() {
            // Only show it if we're connected
            if (this.buffer.getNetwork().state !== 'connected') {
                return false;
            }

            let isCorrectBufferType = (this.buffer.isChannel() || this.buffer.isQuery());
            let isSupported = !!this.buffer.getNetwork().ircClient.network.supports('chathistory');
            return isCorrectBufferType && isSupported && this.buffer.flags.chathistory_available;
        },
        ourNick: function ourNick() {
            return this.buffer ?
                this.buffer.getNetwork().nick :
                '';
        },
    },
    methods: {
        bufferSetting: function bufferSetting(key) {
            return this.buffer.setting(key);
        },
        formatTime: function formatTime(time) {
            return strftime(this.buffer.setting('timestamp_format') || '%T', new Date(time));
        },
        formatMessage: function formatMessage(messageBody) {
            let words = messageBody.split(' ');
            words = words.map(word => {
                let parsed;

                parsed = TextFormatting.linkifyUrls(word, {
                    addHandle: true,
                    handleClass: 'fa fa-chevron-right kiwi-messagelist-message-linkhandle',
                });
                if (parsed !== word) return parsed;

                parsed = TextFormatting.linkifyChannels(word);
                if (parsed !== word) return parsed;

                parsed = TextFormatting.linkifyUsers(word, this.usersAsObject);
                if (parsed !== word) return parsed;

                return _.escape(word);
            });

            let parsed = words.join(' ');
            parsed = TextFormatting.ircCodesToHtml(parsed, this.useExtraFormatting);

            return parsed;
        },
        isMessageHighlight: function isMessageHighlight(message) {
            // Highlighting ourselves when we join or leave a channel is silly
            if (message.type === 'traffic') {
                return false;
            }

            let network = this.buffer.getNetwork();
            if (message.message.toLowerCase().indexOf(network.nick.toLowerCase()) > -1) {
                return true;
            }

            let highlightFound = false;
            (state.setting('highlights') || '').toLowerCase().split(' ').forEach(word => {
                if (!word) {
                    return;
                }

                if (message.message.indexOf(word) > -1) {
                    highlightFound = true;
                }
            });

            return highlightFound;
        },
        nickStyle: function nickColour(nick) {
            return 'color:' + TextFormatting.createNickColour(nick) + ';';
        },
        onThreadClick: function onThreadClick(event) {
            let channelName = event.target.getAttribute('data-channel-name');
            if (channelName) {
                let network = this.buffer.getNetwork();
                state.addBuffer(this.buffer.networkid, channelName);
                network.ircClient.join(channelName);
            }

            let userNick = event.target.getAttribute('data-nick');
            if (userNick) {
                // let buffer = state.addBuffer(this.buffer.networkid, userNick);
                // state.setActiveBuffer(buffer.networkid, buffer.name);
                let user = state.getUser(this.buffer.networkid, userNick);
                if (user) {
                    state.$emit('userbox.show', user, {
                        top: event.clientY,
                        left: event.clientX,
                        buffer: this.buffer,
                    });
                }
            }

            let url = event.target.getAttribute('data-url');
            if (url) {
                state.$emit('mediaviewer.show', url);
            }
        },
        onThreadScroll: function onThreadScroll() {
            let el = this.$el;
            let scrolledUpByPx = el.scrollHeight - (el.offsetHeight + el.scrollTop);

            if (scrolledUpByPx > 30) {
                this.auto_scroll = false;
            } else {
                this.auto_scroll = true;
            }
        },
    },
    watch: {
        buffer: function watchBuffer(newBuffer) {
            if (!newBuffer) {
                return;
            }

            // First time opening this buffer, see if we can load any initial scrollback
            if (!newBuffer.flags.has_opened && this.shouldShowChathistoryTools) {
                newBuffer.requestScrollback();
            }

            if (this.buffer.getNetwork().state === 'connected') {
                newBuffer.flags.has_opened = true;
            }

            this.$el.scrollTop = this.$el.scrollHeight;
        },
        messages: function watchMessages() {
            if (this.auto_scroll) {
                this.$nextTick(() => {
                    this.$el.scrollTop = this.$el.scrollHeight;
                });
            }
        },
    },
};
</script>

<style>
.kiwi-messagelist {
    overflow-y: auto;
}
.kiwi-messagelist-scrollback {
    text-align: center;
    padding: 5px;
}
.kiwi-messagelist-message {
    overflow: hidden;
}
.kiwi-messagelist-nick {
    width: 120px;
    display: inline-block;
    float: left;
}
.kiwi-messagelist-time {
    display: inline-block;
    float: right;
}
.kiwi-messagelist-body {
    display: block;
    white-space: pre-wrap;
    word-wrap: break-word;
    margin-left: 130px;
}
.kiwi-messagelist-body a {
    word-break: break-all;
}
@media screen and (max-width: 700px) {
    .kiwi-messagelist-nick {
        display: inline;
        width: auto;
        float: none;
    }
    .kiwi-messagelist-body {
        margin-left: 2px;
    }
    .kiwi-messagelist-time {
    }
    .kiwi-messagelist-message-repeat .kiwi-messagelist-nick {
        display: none;
    }
}
.kiwi-messagelist-message-traffic .kiwi-messagelist-nick {
    display: none;
}
</style>
=======
<template>
    <div class="kiwi-messagelist" @scroll.self="onThreadScroll" @click.self="onListClick">
        <div
            v-if="shouldShowChathistoryTools"
            class="kiwi-messagelist-scrollback"
        >
            <a @click="buffer.requestScrollback()" class="u-link">{{$t('messages_load')}}</a>
        </div>

        <message-list-message-modern
            v-if="listType === 'modern'"
            v-for="(message, idx) in filteredMessages"
            :message="message"
            :idx="idx"
            :ml="thisMl"
            :key="message.id"
        ></message-list-message-modern>
        <message-list-message-compact
            v-if="listType !== 'modern'"
            v-for="(message, idx) in filteredMessages"
            :message="message"
            :idx="idx"
            :ml="thisMl"
            :key="message.id"
        ></message-list-message-compact>
    </div>
</template>

<script>

import strftime from 'strftime';
import state from 'src/libs/state';
import * as TextFormatting from 'src/helpers/TextFormatting';
import NotConnected from './NotConnected';
import MessageListMessageCompact from './MessageListMessageCompact';
import MessageListMessageModern from './MessageListMessageModern';

// If we're scrolled up more than this many pixels, don't auto scroll down to the bottom
// of the message list
const BOTTOM_SCROLL_MARGIN = 30;

export default {
    components: {
        NotConnected,
        MessageListMessageModern,
        MessageListMessageCompact,
    },
    data: function data() {
        return {
            auto_scroll: true,
            chathistoryAvailable: true,
            hover_nick: '',
            message_info_open: null,
        };
    },
    props: ['buffer', 'messages', 'users'],
    computed: {
        thisMl: function thisMl() {
            return this;
        },
        listType: function listType() {
            return state.setting('messageLayout');
        },
        useExtraFormatting: function useExtraFormatting() {
            // Enables simple markdown formatting
            return this.buffer.setting('extra_formatting');
        },
        filteredMessages: function filteredMessages() {
            let network = this.buffer.getNetwork();
            let currentNick = network.nick;

            let messages = this.messages.slice(0, this.messages.length);
            messages.sort((a, b) => {
                if (a.time > b.time) {
                    return 1;
                } else if (b.time > a.time) {
                    return -1;
                }

                return 0;
            });

            let list = [];
            let maxSize = this.buffer.setting('scrollback_size');
            let showJoinParts = this.buffer.setting('show_joinparts');
            for (let i = messages.length - 1; i >= 0 && list.length < maxSize; i--) {
                if (!showJoinParts && messages[i].type === 'traffic') {
                    continue;
                }
                // Ignored users have the ignore flag set
                if (messages[i].ignore) {
                    continue;
                }

                // When we join a channel the topic is usually sent next. But this looks
                // ugly when rendered. So we switch the topic + join messages around so
                // that the topic is first in the message list.
                if (
                    messages[i].type === 'topic' &&
                    messages[i - 1] &&
                    messages[i - 1].type === 'traffic' &&
                    messages[i - 1].nick === currentNick
                ) {
                    list.push(messages[i - 1]);
                    list.push(messages[i]);
                    i--;
                } else {
                    list.push(messages[i]);
                }
            }

            return list.reverse();
        },
        shouldShowChathistoryTools: function shouldShowChathistoryTools() {
            // Only show it if we're connected
            if (this.buffer.getNetwork().state !== 'connected') {
                return false;
            }

            let isCorrectBufferType = (this.buffer.isChannel() || this.buffer.isQuery());
            let isSupported = !!this.buffer.getNetwork().ircClient.network.supports('chathistory');
            return isCorrectBufferType && isSupported && this.buffer.flags.chathistory_available;
        },
        ourNick: function ourNick() {
            return this.buffer ?
                this.buffer.getNetwork().nick :
                '';
        },
    },
    methods: {
        isHoveringOverMessage: function isHoveringOverMessage(message) {
            return message.nick && message.nick.toLowerCase() === this.hover_nick.toLowerCase();
        },
        toggleMessageInfo: function toggleMessageInfo(message) {
            if (!message) {
                this.message_info_open = null;
            } else if (this.message_info_open === message) {
                return;
            } else if (this.canShowInfoForMessage(message)) {
                // If in the process of selecting text, don't show the info box
                let sel = window.getSelection();
                if (sel.rangeCount > 0) {
                    let range = sel.getRangeAt(0);
                    if (range && !range.collapsed) {
                        return;
                    }
                }

                this.message_info_open = message;
                this.$nextTick(this.maybeScrollToBottom.bind(this));
            }
        },
        canShowInfoForMessage: function canShowInfoForMessage(message) {
            let showInfoForTypes = ['privmsg', 'notice', 'action'];
            return showInfoForTypes.indexOf(message.type) > -1;
        },
        bufferSetting: function bufferSetting(key) {
            return this.buffer.setting(key);
        },
        formatTime: function formatTime(time) {
            return strftime(this.buffer.setting('timestamp_format') || '%T', new Date(time));
        },
        formatMessage: function formatMessage(message) {
            return message.parseHtml(this);
        },
        isMessageHighlight: function isMessageHighlight(message) {
            // Highlighting ourselves when we join or leave a channel is silly
            if (message.type === 'traffic') {
                return false;
            }

            let network = this.buffer.getNetwork();
            if (message.message.toLowerCase().indexOf(network.nick.toLowerCase()) > -1) {
                return true;
            }

            let highlightFound = false;
            (state.setting('highlights') || '').toLowerCase().split(' ').forEach(word => {
                if (!word) {
                    return;
                }

                if (message.message.indexOf(word) > -1) {
                    highlightFound = true;
                }
            });

            return highlightFound;
        },
        nickStyle: function nickColour(nick) {
            return 'color:' + TextFormatting.createNickColour(nick) + ';';
        },
        onListClick: function onListClick(event) {
            this.toggleMessageInfo();
        },
        onMessageClick: function onThreadClick(event, message) {
            let channelName = event.target.getAttribute('data-channel-name');
            if (channelName) {
                let network = this.buffer.getNetwork();
                state.addBuffer(this.buffer.networkid, channelName);
                network.ircClient.join(channelName);
                return;
            }

            let userNick = event.target.getAttribute('data-nick');
            if (userNick) {
                let user = state.getUser(this.buffer.networkid, userNick);
                if (user) {
                    state.$emit('userbox.show', user, {
                        top: event.clientY,
                        left: event.clientX,
                        buffer: this.buffer,
                    });
                }

                return;
            }

            let url = event.target.getAttribute('data-url');
            if (url) {
                state.$emit('mediaviewer.show', url);
            }

            if (this.message_info_open && this.message_info_open !== message) {
                // Clicking on another message while another info is open, just close the info
                this.toggleMessageInfo();
                event.preventDefault();
                return;
            }

            if (this.canShowInfoForMessage(message) && event.target.nodeName === 'A') {
                // Stop links from doing their thing. We show a preview and normal links for that
                event.preventDefault();
            }

            this.toggleMessageInfo(message);
        },
        onThreadScroll: function onThreadScroll() {
            let el = this.$el;
            let scrolledUpByPx = el.scrollHeight - (el.offsetHeight + el.scrollTop);

            if (scrolledUpByPx > BOTTOM_SCROLL_MARGIN) {
                this.auto_scroll = false;
            } else {
                this.auto_scroll = true;
            }
        },
        scrollToBottom: function scrollToBottom() {
            this.$el.scrollTop = this.$el.scrollHeight;
        },
        maybeScrollToBottom: function maybeScrollToBottom() {
            if (this.auto_scroll) {
                this.$el.scrollTop = this.$el.scrollHeight;
            }
        },
    },
    watch: {
        buffer: function watchBuffer(newBuffer) {
            if (!newBuffer) {
                return;
            }

            this.message_info_open = null;

            // First time opening this buffer, see if we can load any initial scrollback
            if (!newBuffer.flags.has_opened && this.shouldShowChathistoryTools) {
                newBuffer.requestScrollback();
            }

            if (this.buffer.getNetwork().state === 'connected') {
                newBuffer.flags.has_opened = true;
            }

            this.scrollToBottom();
        },
        messages: function watchMessages() {
            this.$nextTick(() => {
                this.maybeScrollToBottom();
            });
        },
    },
    mounted: function mounted() {
        this.$nextTick(() => {
            this.scrollToBottom();
        });

        this.listen(state, 'mediaviewer.opened', () => {
            this.$nextTick(this.maybeScrollToBottom.apply(this));
        });
    },
};
</script>

<style>
.kiwi-messagelist {
    overflow-y: auto;
    height: 100%;
}
.kiwi-messagelist-scrollback {
    text-align: center;
    padding: 5px;
}
.kiwi-messagelist-message {
    overflow: hidden;
    transition: opacity 0.2s
}
.kiwi-messagelist-message--blur {
    opacity: 0.5;
}
.kiwi-messagelist-message-traffic .kiwi-messagelist-nick {
    display: none;
}

</style>
>>>>>>> 1427fcb46826c1bd7cbe1d18f6191f74637bdff2
