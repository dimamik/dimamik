---
title: "How I migrated my Google Drive videos from 3GP to MP4"
date: 2025-02-08T00:00:00Z
description: I love keeping memories, but sometimes they take up too much space. In this post, I'll walk you through how I reduced the amount of space my old videos take by about 30%.
tags:
  - general
  - advent of code
categories:
  - General
draft: false
cover:
  image: /img/posts/general/3gp_to_mp4.webp
---

## TL;DR

- I'm using Google Drive to store a bunch of videos, and it started running out of the memory limit, so I needed to find a way to reduce the space.
- 3GP uses H.263 for video encoding and AMR-NB for audio encoding, whereas MP4 uses H.264 for video encoding and AAC for audio encoding. This makes MP4 files occupy ~30% less space.
- I haven't chosen H.265 or AV1 because of compatibility issues. Maybe in the future we'll do another round of migration.
- I've downloaded a Google Drive client for Mac, set up a full sync, and was operating on it like a normal file system.
- It took me 15 minutes to write an Elixir script and 1 hour to run it on all my videos, so please don't judge it too harshly.
- ❗Metadata is the only caveat I was left with, since Google Drive doesn't respect timestamps when syncing files back. For me, it wasn't a big deal, but for some, it might be. So, when I was done, all files looked like new ones (but I had timestamps in their headers, so no big deal for me).

## Space efficiency

- 3GP (H.263) → MP4 (H.264)
  - H.264 is usually 40–50% more efficient than H.263 for the same video quality. That means if your 3GP video used H.263, recoding it properly to H.264 can reduce file size by around 30–50% on average (or keep the same size and achieve better quality).
- 3GP (H.264 Baseline) → MP4 (H.264 Main/High)
  - If your 3GP already uses H.264 but only the baseline profile, migrating to MP4 with H.264 Main or High profile can still yield a 10–20% size reduction. The differences aren’t as dramatic as with older codecs (like H.263), but they are still noticeable.
- 3GP (Any older codec) → MP4 (H.265/HEVC or AV1)
  - Newer codecs like H.265 (HEVC) or AV1 can be 50%+ more efficient than H.264 in ideal conditions. You could see very significant file-size savings compared to old 3GP formats—sometimes 50–70% or more. The trade-off is compatibility: some older devices might not handle H.265/AV1 without special apps or decoders.

## Code

You can run this from a livebook, which can be installed with prepacked elixir and erlang:

[![Run in Livebook](https://livebook.dev/badge/v1/blue.svg)](https://livebook.dev/run?url=https://gist.github.com/dimamik/421398308830e5daf67fa81dbd5bfe02)

```elixir
defmodule GoogleDriveStreamConverter do
  @moduledoc """
  - Lazily traverses google drive path to find .3gp files
  - Processes each file immediately when discovered (no sorting, no concurrency)
  - converts it to .mp4 using ffmpeg, preserves timestamps
  - removes the .3gp file on success.

  NOTE: Uses 'touch -r' (Unix-like) to copy timestamps. On Windows, you may
  need a Unix-like environment (Git Bash, Cygwin, WSL) or adapt the logic.

  **When reuploaded to Google Drive, files loose their timestamps.**
  """

  @drive_path Path.expand("<TODO: your Google Drive path here>")

  def run do
    IO.puts("Scanning lazily for .3gp files in: #{@drive_path}")

    stream_3gp_files(@drive_path)
    |> Stream.each(&process_file/1)
    |> Stream.run()
  end

  defp stream_3gp_files(base_path) do
    # If the base path doesn't exist, return an empty stream
    if not File.exists?(base_path) do
      Stream.map([], & &1)
    else
      Stream.resource(
        fn ->
          [base_path]
        end,
        fn
          [] ->
            {:halt, []}

          [current | rest] ->
            cond do
              File.dir?(current) ->
                case File.ls(current) do
                  {:ok, entries} ->
                    subpaths = Enum.map(entries, &Path.join(current, &1))
                    {[], rest ++ subpaths}

                  {:error, _} ->
                    {[], rest}
                end

              Path.extname(current) == ".3gp" ->
                {[current], rest}

              true ->
                {[], rest}
            end
        end,
        fn _ -> :ok end
      )
    end
  end

  defp process_file(file_path) do
    mp4_path = Path.rootname(file_path) <> ".mp4"

    IO.puts("""
    Found .3gp file:
      - #{file_path}
    Converting to:
      - #{mp4_path}
    """)

    case convert_to_mp4(file_path, mp4_path) do
      :ok ->
        # Copy original timestamps to the new .mp4
        case preserve_timestamps(file_path, mp4_path) do
          :ok ->
            File.rm(file_path)

            IO.puts(
              "✔ Conversion & timestamp preservation successful. Removed original: #{file_path}\n"
            )

          {:error, reason} ->
            IO.puts("⚠ Could not preserve timestamps for #{mp4_path}: #{reason}")
            # Decide if you still remove the original if preserving fails
            File.rm(file_path)
        end

      {:error, reason} ->
        IO.puts("✘ Conversion failed for #{file_path}:\n#{reason}\n")
    end
  end

  defp convert_to_mp4(input_file, output_file) do
    args = [
      # Overwrite output if it exists
      "-y",
      "-i",
      input_file,
      "-c:v",
      "libx264",
      "-preset",
      "slow",
      "-crf",
      "23",
      "-c:a",
      "aac",
      "-b:a",
      "128k",
      output_file
    ]

    {output, exit_code} = System.cmd("/opt/homebrew/bin/ffmpeg", args, stderr_to_stdout: true)

    if exit_code == 0 do
      :ok
    else
      {:error, output}
    end
  end

  defp preserve_timestamps(original, new_file) do
    case System.cmd("touch", ["-r", original, new_file]) do
      {_, 0} ->
        :ok

      {error_msg, code} ->
        {:error, "touch exited with code #{code}: #{error_msg}"}
    end
  end
end

GoogleDriveStreamConverter.run()
```
