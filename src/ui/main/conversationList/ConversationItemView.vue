<template>
    <div class="conversation-item-container"
         @dragover="dragEvent($event, 'dragover')"
         @dragleave="dragEvent($event, 'dragleave')"
         @dragenter="dragEvent($event,'dragenter')"
         @drop="dragEvent($event, 'drop')"
         @click="showConversation"
         v-bind:class="{
             drag: dragAndDropEnterCount > 0,
             active: shareConversationState.currentConversationInfo && shareConversationState.currentConversationInfo.conversation.equal(source.conversation),
                              top:source.top,
                              highlight:shareConversationState.contextMenuConversationInfo && shareConversationState.contextMenuConversationInfo.conversation.equal(source.conversation)
         }"
         @contextmenu.prevent="showConversationInfoContextMenu">
        <div class="conversation-item">
            <div class="header">
                <img class="avatar" draggable="false" :src="portrait" alt=""
                     @error="imgUrlAlt"/>
                <em v-if="unread > 0" class="badge" v-bind:class="{silent:source.isSilent}">{{ unread }}</em>
            </div>
            <div class="content-container">
                <div class="title-time-container">
                    <i v-if="source.conversation.type === 5" class="icon-ion-android-lock" style="padding-right: 5px"></i>
                    <h2 class="title single-line">{{ conversationTitle }}</h2>
                    <p class="time">{{ source._timeStr }}</p>
                </div>
                <div class="content">
                    <p class="draft single-line" v-if="shouldShowDraft" v-html="draft"></p>
                    <p class="draft single-line" v-else-if="shouldShowVoipStatus" v-html="voipOngoingDesc"></p>
                    <p class="last-message-desc single-line" v-else>
                        <i v-if="unreadMention > 0">[有人@我]</i>
                        {{ lastMessageContent }}
                    </p>
                    <i v-if="source.isSilent" class="icon-ion-android-notifications-off"></i>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import store from "@/store";
import Draft from "@/ui/util/draft";
import FileMessageContent from "@/wfc/messages/fileMessageContent";
import Message from "@/wfc/messages/message";
import wfc from "@/wfc/client/wfc";
import NotificationMessageContent from "@/wfc/messages/notification/notificationMessageContent";
import Config from "../../../config";
import {getConversationPortrait} from "../../util/imageUtil";
import ConversationType from "../../../wfc/model/conversationType";

export default {
    name: "ConversationItemView",
    props: {
        source: {
            type: Object,
            required: true,
        },
    },
    data() {
        return {
            dragAndDropEnterCount: 0,
            shareConversationState: store.state.conversation,
            groupPortrait: Config.DEFAULT_GROUP_PORTRAIT_URL,
        };
    },
    mounted() {
        this.refreshGroupPortrait();
    },
    methods: {
        dragEvent(e, v) {
            if (v === 'dragenter') {
                this.dragAndDropEnterCount++;
            } else if (v === 'dragleave') {
                this.dragAndDropEnterCount--;
            } else if (v === 'drop') {
                this.dragAndDropEnterCount--;
                let length = e.dataTransfer.files.length;
                if (length > 0 && length < 5) {
                    for (let i = 0; i < length; i++) {
                        this.$eventBus.$emit('uploadFile', e.dataTransfer.files[i])
                        store.sendFile(this.source.conversation, e.dataTransfer.files[i]);
                    }
                } else {
                    // TODO
                    let url = e.dataTransfer.getData('URL');
                    if (url) {
                        store.sendFile(this.source.conversation, url);
                    } else {
                        let text = e.dataTransfer.getData('text');
                        if (text.startsWith('{')) {
                            let obj = JSON.parse(text);
                            let file = new FileMessageContent(null, obj.url, obj.name, obj.size)
                            let message = new Message(this.source.conversation, file)
                            wfc.sendMessage(message);
                        }
                    }
                    console.log(this.$t('conversation.drag_to_send_limit_tip'), e.dataTransfer, e.dataTransfer.getData('URL'));
                }
            } else if (v === 'dragover') {
                // If not st as 'copy', electron will open the drop file
                e.dataTransfer.dropEffect = 'copy';
            }
        },
        imgUrlAlt(e) {
            if (this.source.conversation.type === ConversationType.Group) {
                e.target.src = Config.DEFAULT_GROUP_PORTRAIT_URL;
            } else {
            	e.target.src = Config.DEFAULT_PORTRAIT_URL;
            }
        },

        showConversation() {
            store.setCurrentConversationInfo(this.source);
            if (this.unread > 0) {
            	wfc.clearConversationUnreadStatus(this.source.conversation);
            }
            this.refreshGroupPortrait();
        },
        showConversationInfoContextMenu(event) {
            this.$eventBus.$emit('showConversationContextMenu', event, this.source);
        },

        refreshGroupPortrait() {
            let info = this.source;
            if (info.conversation.type !== ConversationType.Group) {
                return;
            }
            console.log('refreshGroupPortrait', !info.conversation._target.portrait, info.conversation._target.portrait === Config.DEFAULT_GROUP_PORTRAIT_URL);
            if (!info.conversation._target.portrait || info.conversation._target.portrait === Config.DEFAULT_GROUP_PORTRAIT_URL) {
                getConversationPortrait(info.conversation).then((portrait => {
                    if (info.conversation.equal(this.source.conversation)) {
                        console.log('update portrait', this.source.conversation.target)
                        if (portrait !== Config.DEFAULT_GROUP_PORTRAIT_URL){
                            info.conversation._target.portrait = portrait;
                        }
                        this.groupPortrait = portrait;
                    }
                }))
            }
        }
    },
    computed: {
        conversationTitle() {
            let info = this.source;
            if (info.conversation._target) {
                return info.conversation._target._displayName;
            }
            return '';
        },

        shouldShowDraft() {
            if (this.shareConversationState.currentConversationInfo && this.shareConversationState.currentConversationInfo.conversation.equal(this.source.conversation)) {
                return false;
            }
            if (this.source.unreadCount.unreadMention + this.source.unreadCount.unreadMentionAll > 0) {
                return false;
            }
            let draft = Draft.getConversationDraftEx(this.source);
            return draft.text.trim() !== '' || draft.quotedMessage !== null;
        },

        shouldShowVoipStatus() {
            return this.source._isVoipOngoing;
        },

        draft() {
            let draft = Draft.getConversationDraftEx(this.source);
            let draftText = `<em>[${this.$t('common.draft')}]</em>` + draft.text;
            draftText = draftText.replace(/<img [:a-zA-Z0-9_+; ,\-=\/."]+>/g, '[图片]')
            draftText = draftText.replace(/&nbsp;/g, ' ');
            draftText = draftText.replace(/<br>/g, '')
            if (draft.quotedMessage) {
                draftText += '...'
            }
            return draftText;
        },

        voipOngoingDesc() {
            let voipStatus = `<em>[音视频通话进行中]</em>`;
            return voipStatus;
        },

        lastMessageContent() {
            let conversationInfo = this.source;
            if (conversationInfo.lastMessage && conversationInfo.lastMessage.messageContent) {
                let senderName = '';
                if (conversationInfo.conversation.type === 1 && conversationInfo.lastMessage.direction === 1 && !(conversationInfo.lastMessage.messageContent instanceof NotificationMessageContent)) {
                    if (conversationInfo.lastMessage._from) {
                        senderName = conversationInfo.lastMessage._from._displayName + ': ';
                    } else {
                        conversationInfo.lastMessage = store._patchMessage(conversationInfo.lastMessage, 0)
                        if (conversationInfo.lastMessage._from) {
                            senderName = conversationInfo.lastMessage._from._displayName + ': ';
                        } else {
                            senderName = '<' + conversationInfo.lastMessage.from + '>: ';
                        }
                    }
                }
                return senderName + conversationInfo.lastMessage.messageContent.digest(conversationInfo.lastMessage);
            } else {
                return '';
            }
        },

        unread() {
            let conversationInfo = this.source;
            let unreadCount = conversationInfo.unreadCount;
            return unreadCount ? (unreadCount.unread + unreadCount.unreadMention + unreadCount.unreadMentionAll) : 0;
        },
        unreadMention() {
            let conversationInfo = this.source;
            let unreadCount = conversationInfo.unreadCount;
            return unreadCount ? (unreadCount.unreadMention + unreadCount.unreadMentionAll) : 0;
        },

        portrait() {
            let info = this.source;
            if (info.conversation.type === ConversationType.Group) {
                if (info.conversation._target.portrait) {
                    return info.conversation._target.portrait;
                } else {
                    return this.groupPortrait;
                }
            } else {
                return info.conversation._target.portrait;
            }
        }
    },
};
</script>

<style scoped>
.conversation-item-container {
    padding-left: 12px;
    background-color: #f8f8f8;
}

.conversation-item-container.drag {
    border: 1px solid #4168e0;
}

.conversation-item-container.active {
    background-color: #d6d6d6;
}

.conversation-item-container.top {
    background-color: #f1f1f1;
}

.conversation-item-container.highlight {
    box-shadow: 0 0 0 2px #4168e0 inset;
    z-index: 100;
}

.conversation-item-container.active.top {
    background-color: #d6d6d6;
}

.conversation-item {
    width: 100%;
    height: 70px;
    display: flex;
    /*border-bottom: 1px solid #eeeeee;*/
    align-items: center;
    justify-content: center;
}

.header {
    height: 100%;
    padding: 10px 12px 10px 0;
    margin-right: 2px;
    position: relative;
}

.header .avatar {
    position: relative;
    width: 45px;
    height: 45px;
    min-width: 45px;
    min-height: 45px;
    background: #d6d6d6;
    top: 50%;
    transform: translateY(-50%);
    border-radius: 3px;
}

.header .badge {
    position: absolute;
    color: white;
    font-size: 10px;
    background-color: red;
    border-radius: 8px;
    width: 16px;
    height: 16px;
    line-height: 16px;
    font-style: normal;
    text-align: center;
    right: 8px;
    top: 8px;
}

.header .badge.silent {
    width: 8px;
    height: 8px;
    font-size: 0;
}

.content-container {
    width: 100%;
    height: 50px;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    justify-content: space-around;
    padding-right: 12px;
}

.content-container .title-time-container {
    display: flex;
    width: 100%;
    align-content: center;
    justify-content: space-between;
}

.content-container .title-time-container .title {
    display: inline-block;
    font-size: 14px;
    color: #262626;
    font-style: normal;
    font-weight: normal;
    padding-right: 10px;
    flex: 1;
}

.content-container .title-time-container .time {
    display: inline-block;
    color: gray;
    font-size: 10px;
}

.content-container .content {
    display: flex;
    justify-content: space-between;
}

.content .draft {
    font-size: 13px;
    height: 20px;
}

/*refer to: https://blog.csdn.net/weixin_42412046/article/details/80804285*/
>>> .content .draft em {
    color: red;
    font-style: normal;
    padding-right: 5px;
}

.content .last-message-desc {
    color: #b8b8b8;
    font-size: 13px;
}

.content .last-message-desc i {
    font-style: normal;
    color: red;
}

.content i {
    color: #b8b8b8;
}


</style>
