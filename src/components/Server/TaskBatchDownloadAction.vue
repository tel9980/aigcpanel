<script setup lang="ts">
import {Dialog} from "../../lib/dialog";
import {t} from "../../lang";
import {TaskRecord} from "../../service/TaskService";
import {computed} from "vue";
import {sleep} from "../../lib/util";
import {mapError} from "../../lib/error";

const props = defineProps<{
    records: TaskRecord[],
}>()

const canDownload = computed(() => {
    return props.records.filter(record => record.status === 'success').length > 0
})

const doDownload = async () => {
    const recordsDownload = props.records.filter(record => record.status === 'success')
    if (recordsDownload.length === 0) {
        Dialog.tipError(t('没有可以下载的记录'))
        return
    }
    const pathDir = await window.$mapi.file.openDirectory()
    if (!pathDir) {
        return
    }
    Dialog.loadingOn(t('正在下载'))
    let errors: any = []
    for (const r of recordsDownload) {
        Dialog.loadingUpdate(t('正在下载 {index}/{total}', {
            index: recordsDownload.indexOf(r) + 1,
            total: recordsDownload.length
        }))
        let fromPath = r.result.url
        let targetPath: string = ''
        if (r.biz === 'SoundTts' || r.biz === 'SoundClone') {
            targetPath = `${pathDir}/${r.title}.wav`
        } else if (r.biz === 'VideoGen' || r.biz === 'VideoGenFlow') {
            targetPath = `${pathDir}/${r.title}.mp4`
        }
        await sleep(100)
        if (await window.$mapi.file.exists(targetPath, {
            isFullPath: true,
        })) {
            continue
        }
        try {
            await window.$mapi.file.copy(fromPath, targetPath, {
                isFullPath: true,
            })
        } catch (e) {
            errors.push(mapError(e))
        }
    }
    Dialog.loadingOff()
    if (errors.length > 0) {
        Dialog.tipError(t('下载失败') + '\n' + errors.join('\n'))
        return
    }
    Dialog.tipSuccess(t('下载成功'))
}
</script>

<template>
    <a-tooltip :content="$t('下载')" mini>
        <a-button class="mr-2"
                  :disabled="!canDownload"
                  @click="doDownload()">
            <template #icon>
                <icon-download/>
            </template>
        </a-button>
    </a-tooltip>
</template>
