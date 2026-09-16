<script lang="ts">
  import { onMount } from 'svelte'
  import Icon from '@iconify/svelte'
  import * as Select from '$lib/components/ui/select'
  import { Label } from '$lib/components/ui/label'
  import Switch from '$lib/components/ui/switch/Switch.svelte'
  import RecipientInput from '$lib/components/composer/RecipientInput.svelte'
  import { accountStore } from '$lib/stores/accounts.svelte'
  // @ts-ignore - Wails generated imports
  import { smtp } from '../../../../wailsjs/go/models'
  // @ts-ignore - wailsjs path
  import { GetDefaultBcc, GetDefaultBccEnabled, SetDefaultBcc, SetDefaultBccEnabled } from '../../../../wailsjs/go/app/App.js'
  import { _ } from '$lib/i18n'
  import { supportedLocales } from '$lib/i18n'
  import { SPELLCHECK_DICTS } from '$lib/spellcheck/locales'
  import { getSpellcheckCustomWords } from '$lib/stores/settings.svelte'
  import { removeCustomWord } from '$lib/spellcheck/settings'
  import SpellWordList from '$lib/spellcheck/SpellWordList.svelte'

  interface Props {
    composerMode: string
    mailtoMode: string
    composerFormat: string
    readReceiptResponsePolicy: string
    spellcheckEnabled: boolean
    spellcheckLanguages: string[]
    onComposerModeChange: (value: string) => void
    onMailtoModeChange: (value: string) => void
    onFormatChange: (value: string) => void
    onPolicyChange: (value: string) => void
    onSpellcheckEnabledChange: (value: boolean) => void
    onSpellcheckLanguagesChange: (value: string[]) => void
  }

  let {
    composerMode = $bindable(),
    mailtoMode = $bindable(),
    composerFormat = $bindable(),
    readReceiptResponsePolicy = $bindable(),
    spellcheckEnabled = $bindable(),
    spellcheckLanguages = $bindable(),
    onComposerModeChange,
    onMailtoModeChange,
    onFormatChange,
    onPolicyChange,
    onSpellcheckEnabledChange,
    onSpellcheckLanguagesChange,
  }: Props = $props()

  // Dictionaries Aerion ships, with their native display names.
  const dictLanguages = SPELLCHECK_DICTS.map((code) => ({
    code,
    name: supportedLocales.find((l) => l.code === code)?.name ?? code,
  }))

  // Added-words list — read straight from the store (persists immediately on
  // add/remove, independent of the dialog's Save flow).
  const addedWords = $derived(getSpellcheckCustomWords())

  const modeOptions = $derived([
    { value: 'inline', label: $_('settings.composerModeInline') },
    { value: 'detached', label: $_('settings.composerModeDetached') },
  ])

  const formatOptions = $derived([
    { value: 'rich', label: $_('settings.composerFormatRich') },
    { value: 'plain', label: $_('settings.composerFormatPlain') },
  ])

  const readReceiptResponseOptions = $derived([
    { value: 'never', label: $_('settingsGeneral.neverSendReceipts') },
    { value: 'ask', label: $_('settingsGeneral.askEachTime') },
    { value: 'always', label: $_('settingsGeneral.alwaysSendReceipts') },
  ])

  function getModeLabel(mode: string): string {
    return modeOptions.find(o => o.value === mode)?.label ?? mode
  }

  function getPolicyLabel(value: string): string {
    return readReceiptResponseOptions.find(o => o.value === value)?.label ?? value
  }

  function handleComposerModeChange(value: string | undefined) {
    if (!value) return
    composerMode = value
    onComposerModeChange?.(value)
  }

  function handleMailtoModeChange(value: string | undefined) {
    if (!value) return
    mailtoMode = value
    onMailtoModeChange?.(value)
  }

  function getFormatLabel(value: string): string {
    return formatOptions.find(o => o.value === value)?.label ?? value
  }

  function handleFormatChange(value: string | undefined) {
    if (!value) return
    composerFormat = value
    onFormatChange?.(value)
  }

  function handlePolicyChange(value: string | undefined) {
    if (!value) return
    readReceiptResponsePolicy = value
    onPolicyChange?.(value)
  }

  function handleSpellcheckEnabledChange(value: boolean) {
    spellcheckEnabled = value
    onSpellcheckEnabledChange?.(value)
  }

  function handleLangToggle(code: string, on: boolean) {
    const next = on
      ? [...spellcheckLanguages, code]
      : spellcheckLanguages.filter((c) => c !== code)
    spellcheckLanguages = next
    onSpellcheckLanguagesChange?.(next)
  }

  // Per-account default BCC (#341). Values save directly (no dialog-level
  // Save plumbing); a disabled account keeps its saved value for re-enable.
  // Fields reuse the composer's RecipientInput so addresses become chips
  // with the same autocomplete/paste behavior; stored as a comma-joined
  // string in settings.
  const bccAccounts = $derived(accountStore.accounts.filter(acc => !acc.account.sharedMailboxParentId))
  let bccEnabled = $state<Record<string, boolean>>({})
  let bccAddresses = $state<Record<string, smtp.Address[]>>({})
  const bccLastSaved: Record<string, string> = {}

  function parseBccList(raw: string): smtp.Address[] {
    return raw
      .split(/[,;]/)
      .map(s => s.trim())
      .filter(Boolean)
      .map(addr => new smtp.Address({ name: '', address: addr }))
  }

  onMount(async () => {
    for (const acc of accountStore.accounts) {
      const id = acc.account.id
      try {
        bccEnabled[id] = await GetDefaultBccEnabled(id)
        const raw = await GetDefaultBcc(id)
        bccLastSaved[id] = raw
        bccAddresses[id] = parseBccList(raw)
      } catch (err) {
        console.error('Failed to load default BCC settings:', err)
      }
    }
  })

  async function handleBccToggle(accountId: string, on: boolean) {
    bccEnabled[accountId] = on
    try {
      await SetDefaultBccEnabled(accountId, on)
      if (on) {
        const raw = await GetDefaultBcc(accountId)
        bccLastSaved[accountId] = raw
        bccAddresses[accountId] = parseBccList(raw)
      }
    } catch (err) {
      console.error('Failed to save default BCC toggle:', err)
    }
  }

  // Persist chip changes as they happen (RecipientInput mutates the bound
  // array on add/remove; no blur event to hook)
  $effect(() => {
    for (const [id, addrs] of Object.entries(bccAddresses)) {
      const serialized = addrs.map(a => a.address).filter(Boolean).join(', ')
      if (serialized === bccLastSaved[id]) continue
      bccLastSaved[id] = serialized
      SetDefaultBcc(id, serialized).catch((err: unknown) => {
        console.error('Failed to save default BCC:', err)
      })
    }
  })
</script>

<div class="space-y-6 p-1">
  <div class="space-y-2">
    <Label>{$_('settings.composerMode')}</Label>
    <Select.Root value={composerMode} onValueChange={handleComposerModeChange}>
      <Select.Trigger>
        <Select.Value placeholder={$_('settings.composerMode')}>
          {getModeLabel(composerMode)}
        </Select.Value>
      </Select.Trigger>
      <Select.Content>
        {#each modeOptions as opt (opt.value)}
          <Select.Item value={opt.value} label={opt.label} />
        {/each}
      </Select.Content>
    </Select.Root>
    <p class="text-xs text-muted-foreground">
      {$_('settings.composerModeDescription')}
    </p>
  </div>

  <div class="space-y-2">
    <Label>{$_('settings.mailtoMode')}</Label>
    <Select.Root value={mailtoMode} onValueChange={handleMailtoModeChange}>
      <Select.Trigger>
        <Select.Value placeholder={$_('settings.mailtoMode')}>
          {getModeLabel(mailtoMode)}
        </Select.Value>
      </Select.Trigger>
      <Select.Content>
        {#each modeOptions as opt (opt.value)}
          <Select.Item value={opt.value} label={opt.label} />
        {/each}
      </Select.Content>
    </Select.Root>
    <p class="text-xs text-muted-foreground">
      {$_('settings.mailtoModeDescription')}
    </p>
  </div>

  <div class="space-y-2">
    <Label>{$_('settings.composerFormat')}</Label>
    <Select.Root value={composerFormat} onValueChange={handleFormatChange}>
      <Select.Trigger>
        <Select.Value placeholder={$_('settings.composerFormat')}>
          {getFormatLabel(composerFormat)}
        </Select.Value>
      </Select.Trigger>
      <Select.Content>
        {#each formatOptions as opt (opt.value)}
          <Select.Item value={opt.value} label={opt.label} />
        {/each}
      </Select.Content>
    </Select.Root>
    <p class="text-xs text-muted-foreground">
      {$_('settings.composerFormatDescription')}
    </p>
  </div>

  <!-- Divider -->
  <div class="border-t border-border"></div>

  <!-- Read Receipts Section -->
  <div class="space-y-4">
    <h3 class="text-sm font-medium flex items-center gap-2">
      <Icon icon="mdi:email-check-outline" class="w-4 h-4" />
      {$_('settingsGeneral.readReceipts')}
    </h3>

    <div class="space-y-2">
      <Label>{$_('settingsGeneral.readReceiptPolicy')}</Label>
      <Select.Root value={readReceiptResponsePolicy} onValueChange={handlePolicyChange}>
        <Select.Trigger>
          <Select.Value placeholder={$_('settingsGeneral.selectPolicy')}>
            {getPolicyLabel(readReceiptResponsePolicy)}
          </Select.Value>
        </Select.Trigger>
        <Select.Content>
          {#each readReceiptResponseOptions as opt (opt.value)}
            <Select.Item value={opt.value} label={opt.label} />
          {/each}
        </Select.Content>
      </Select.Root>
      <p class="text-xs text-muted-foreground">
        {$_('settingsGeneral.readReceiptPolicyHelp')}
      </p>
    </div>
  </div>

  <!-- Divider -->
  <div class="border-t border-border"></div>

  <!-- Spellcheck Section -->
  <div class="space-y-4">
    <h3 class="text-sm font-medium flex items-center gap-2">
      <Icon icon="mdi:spellcheck" class="w-4 h-4" />
      {$_('spellcheck.title')}
    </h3>

    <div class="flex items-center justify-between">
      <div>
        <Label for="spellcheck-enabled">{$_('spellcheck.enable')}</Label>
        <p class="text-xs text-muted-foreground">{$_('spellcheck.enableHelp')}</p>
      </div>
      <Switch id="spellcheck-enabled" checked={spellcheckEnabled} onCheckedChange={handleSpellcheckEnabledChange} />
    </div>

    <div class="space-y-2 {spellcheckEnabled ? '' : 'opacity-50 pointer-events-none'}">
      <Label>{$_('spellcheck.languages')}</Label>
      <div class="space-y-1 max-h-48 overflow-y-auto rounded-md border border-border p-2">
        {#each dictLanguages as lang (lang.code)}
          <div class="flex items-center justify-between px-1 py-1">
            <span class="text-sm">{lang.name}</span>
            <Switch
              id={`spellcheck-lang-${lang.code}`}
              checked={spellcheckLanguages.includes(lang.code)}
              onCheckedChange={(v) => handleLangToggle(lang.code, v)}
            />
          </div>
        {/each}
      </div>
      <p class="text-xs text-muted-foreground">{$_('spellcheck.languagesHelp')}</p>
    </div>

    <SpellWordList
      title={$_('spellcheck.addedWords')}
      words={addedWords}
      emptyText={$_('spellcheck.noAddedWords')}
      removeLabel={$_('spellcheck.removeWord')}
      onRemove={removeCustomWord}
    />
  </div>

  <!-- Divider -->
  <div class="border-t border-border"></div>

  <!-- Default BCC Section (#341) -->
  <div class="space-y-4">
    <h3 class="text-sm font-medium flex items-center gap-2">
      <Icon icon="mdi:email-plus-outline" class="w-4 h-4" />
      {$_('settings.defaultBcc')}
    </h3>

    <div class="space-y-1 max-h-60 overflow-y-auto rounded-md border border-border p-2">
      {#each bccAccounts as acc (acc.account.id)}
        <div class="px-1 py-1 space-y-2">
          <div class="flex items-center justify-between gap-2">
            <span class="text-sm truncate">{acc.account.name} — {acc.account.email}</span>
            <Switch
              id={`default-bcc-${acc.account.id}`}
              checked={bccEnabled[acc.account.id] ?? false}
              onCheckedChange={(v) => handleBccToggle(acc.account.id, v)}
            />
          </div>
          {#if bccEnabled[acc.account.id]}
            <RecipientInput
              bind:recipients={
                () => bccAddresses[acc.account.id] ?? [],
                (v) => { bccAddresses[acc.account.id] = v }
              }
              placeholder="crm@example.com"
            />
          {/if}
        </div>
      {/each}
    </div>
    <p class="text-xs text-muted-foreground">{$_('settings.defaultBccHelp')}</p>
  </div>
</div>
