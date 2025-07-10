"use client"
import type React from "react"
import { useState, useEffect, useRef, useCallback } from "react"
import { Button } from "@/components/ui/button"
import { Label } from "@/components/ui/label"
import { Input } from "@/components/ui/input"
import { Plus, Pause, RotateCcw, Settings, Trash2, Volume2, VolumeX, Music } from "lucide-react"
import { Play } from "lucide-react"

interface Heart {
  id: number
  x: number
  y: number
  speed: number
  rotation: number
  scale: number
  pulse: number
}

interface Particle {
  id: number
  x: number
  y: number
  vx: number
  vy: number
  life: number
  maxLife: number
  rotation: number
  scale: number
}

interface Firework {
  id: number
  x: number
  y: number
  particles: {
    x: number
    y: number
    vx: number
    vy: number
    life: number
    maxLife: number
    color: string
  }[]
}

interface AnimatedElement {
  id: number
  x: number
  y: number
  vx: number
  vy: number
  rotation: number
  rotationSpeed: number
  scale: number
  scaleSpeed: number
  opacity: number
  life: number
  maxLife: number
  type: "heart" | "star" | "sparkle" | "ring" | "text"
  color: string
  text?: string
}

export default function Component() {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const animationRef = useRef<number>()

  const girlImageRef = useRef<HTMLImageElement>(null)
  const [imageLoaded, setImageLoaded] = useState(false)

  // GAME STATE
  const [gameState, setGameState] = useState<"menu" | "playing" | "paused" | "settings" | "victory" | "animation">(
    "menu",
  )
  const [score, setScore] = useState(0)
  const [hearts, setHearts] = useState<Heart[]>([])
  const [particles, setParticles] = useState<Particle[]>([])
  const [fireworks, setFireworks] = useState<Firework[]>([])
  const [girlPosition, setGirlPosition] = useState(50)
  const [showLoveMessage, setShowLoveMessage] = useState(false)
  const [currentMessage, setCurrentMessage] = useState("")
  const [isMovingLeft, setIsMovingLeft] = useState(false)
  const [isMovingRight, setIsMovingRight] = useState(false)

  // VICTORY ANIMATION STATE
  const [victoryAnimationTime, setVictoryAnimationTime] = useState(0)
  const [showVictoryMessage, setShowVictoryMessage] = useState(false)
  const [girlCelebrationFrame, setGirlCelebrationFrame] = useState(0)

  // NEW INTEGRATED ANIMATION STATE
  const [animatedElements, setAnimatedElements] = useState<AnimatedElement[]>([])
  const [animationPhase, setAnimationPhase] = useState(0) // 0: hearts, 1: celebration, 2: message, 3: finale
  const [animationStartTime, setAnimationStartTime] = useState(0)
  const animationElementId = useRef(0)

  // SOUND STATE
  const [soundEnabled, setSoundEnabled] = useState(true)

  // BACKGROUND MUSIC STATE - IMPROVED WITH ACTUAL AUDIO
  const [backgroundMusic, setBackgroundMusic] = useState<HTMLAudioElement | null>(null)
  const [musicPlaying, setMusicPlaying] = useState(false)
  const [musicVolume, setMusicVolume] = useState(0.3)
  const [audioInitialized, setAudioInitialized] = useState(false)
  const [musicEnabled, setMusicEnabled] = useState(true)

  // ANIMATION STATE
  const [animationTime, setAnimationTime] = useState(0)
  const [girlAnimationFrame, setGirlAnimationFrame] = useState(0)
  const [backgroundOffset, setBackgroundOffset] = useState(0)

  // GAME SETTINGS
  const [targetScore, setTargetScore] = useState(50) // Target hearts to collect
  const [newTargetScore, setNewTargetScore] = useState("50")

  // Editable messages
  const [loveMessages, setLoveMessages] = useState([
    "You're my sunshine! ☀️💕",
    "Every heartbeat is for you! 💓",
    "You make my world complete! 🌍💖",
    "Forever and always yours! 💍✨",
    "You're my happy place! 🏠💕",
    "You light up my world! ✨💖",
    "My heart skips for you! 💓🎵",
    "You're my dream come true! 🌙💕",
  ])
  const [newMessage, setNewMessage] = useState("")

  // VIDEO AND FINAL SCREEN STATE
  const [showVideo, setShowVideo] = useState(false)
  const [videoEnded, setVideoEnded] = useState(false)
  const [showFinalMessage, setShowFinalMessage] = useState(false)
  const [heartBurstActive, setHeartBurstActive] = useState(false)
  const videoRef = useRef<HTMLVideoElement>(null)

  // ADD THESE NEW VIDEO UPLOAD STATES:
  const [uploadedVideoUrl, setUploadedVideoUrl] = useState<string | null>(null)
  const [showVideoUpload, setShowVideoUpload] = useState(false)
  const [videoUploading, setVideoUploading] = useState(false)
  const fileInputRef = useRef<HTMLInputElement>(null)

  // ADD THESE NEW CHARACTER UPLOAD STATES:
  const [uploadedCharacterUrl, setUploadedCharacterUrl] = useState<string | null>(null)
  const [characterUploading, setCharacterUploading] = useState(false)
  const characterFileInputRef = useRef<HTMLInputElement>(null)

  // ADD THESE NEW AUDIO STATES:
  const [showAudioPrompt, setShowAudioPrompt] = useState(false)
  const [videoMuted, setVideoMuted] = useState(false)

  // ADD THESE NEW MUSIC UPLOAD STATES:
  const [uploadedMusicUrl, setUploadedMusicUrl] = useState<string | null>(null)
  const [musicUploading, setMusicUploading] = useState(false)
  const musicFileInputRef = useRef<HTMLInputElement>(null)
  const uploadedMusicRef = useRef<HTMLAudioElement | null>(null)

  const gameSpeed = useRef(1)
  const lastHeartSpawn = useRef(0)
  const heartIdCounter = useRef(0)
  const particleIdCounter = useRef(0)
  const fireworkIdCounter = useRef(0)

  const CANVAS_WIDTH = 350
  const CANVAS_HEIGHT = 700

  // SIMPLE SOUND FUNCTIONS - MOVED UP TO AVOID DEPENDENCY ISSUES
  const playSound = useCallback(
    (frequency: number, duration: number, type: "catch" | "message" = "catch") => {
      if (!soundEnabled) return
      try {
        const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)()
        const oscillator = audioContext.createOscillator()
        const gainNode = audioContext.createGain()
        oscillator.connect(gainNode)
        gainNode.connect(audioContext.destination)

        if (type === "catch") {
          oscillator.frequency.setValueAtTime(frequency, audioContext.currentTime)
          oscillator.frequency.exponentialRampToValueAtTime(frequency * 2, audioContext.currentTime + duration)
          gainNode.gain.setValueAtTime(0.1, audioContext.currentTime)
          gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + duration)
        } else {
          oscillator.frequency.setValueAtTime(frequency, audioContext.currentTime)
          oscillator.frequency.setValueAtTime(frequency * 1.5, audioContext.currentTime + duration * 0.3)
          oscillator.frequency.setValueAtTime(frequency * 2, audioContext.currentTime + duration * 0.6)
          gainNode.gain.setValueAtTime(0.15, audioContext.currentTime)
          gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + duration)
        }

        oscillator.type = "sine"
        oscillator.start(audioContext.currentTime)
        oscillator.stop(audioContext.currentTime + duration)
      } catch (error) {
        console.log("Sound play failed:", error)
      }
    },
    [soundEnabled],
  )

  const playCatchSound = useCallback(() => {
    playSound(800, 0.2, "catch")
  }, [playSound])

  const playMessageSound = useCallback(() => {
    playSound(600, 0.8, "message")
  }, [playSound])

  // VICTORY SOUND
  const playVictorySound = useCallback(() => {
    if (!soundEnabled) return
    try {
      const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)()
      // Play a victory melody
      const frequencies = [523, 659, 784, 1047] // C, E, G, C (major chord)
      frequencies.forEach((freq, index) => {
        setTimeout(() => {
          const oscillator = audioContext.createOscillator()
          const gainNode = audioContext.createGain()
          oscillator.connect(gainNode)
          gainNode.connect(audioContext.destination)
          oscillator.frequency.setValueAtTime(freq, audioContext.currentTime)
          gainNode.gain.setValueAtTime(0.1, audioContext.currentTime)
          gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.5)
          oscillator.type = "sine"
          oscillator.start(audioContext.currentTime)
          oscillator.stop(audioContext.currentTime + 0.5)
        }, index * 200)
      })
    } catch (error) {
      console.log("Victory sound play failed:", error)
    }
  }, [soundEnabled])

  // BACKGROUND MUSIC FUNCTIONS - REORGANIZED TO FIX DEPENDENCY ISSUES
  const createGeneratedMusic = useCallback(() => {
    try {
      // Create a simple romantic melody using data URL
      const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)()

      // Create a simple romantic tune
      const createTone = (frequency: number, duration: number, startTime: number) => {
        const oscillator = audioContext.createOscillator()
        const gainNode = audioContext.createGain()

        oscillator.connect(gainNode)
        gainNode.connect(audioContext.destination)

        oscillator.frequency.setValueAtTime(frequency, startTime)
        oscillator.type = "sine"

        gainNode.gain.setValueAtTime(0, startTime)
        gainNode.gain.linearRampToValueAtTime(musicVolume * 0.1, startTime + 0.1)
        gainNode.gain.linearRampToValueAtTime(musicVolume * 0.05, startTime + duration - 0.1)
        gainNode.gain.linearRampToValueAtTime(0, startTime + duration)

        oscillator.start(startTime)
        oscillator.stop(startTime + duration)
      }

      // Play romantic melody
      const playMelody = () => {
        if (!musicEnabled || !musicPlaying) return

        const currentTime = audioContext.currentTime
        const noteDuration = 0.8

        // Simple romantic melody: C-E-G-C-A-F-G-C
        const melody = [261.63, 329.63, 392.0, 523.25, 440.0, 349.23, 392.0, 523.25]

        melody.forEach((freq, index) => {
          createTone(freq, noteDuration, currentTime + index * noteDuration)
        })

        // Add harmony
        const harmony = [130.81, 164.81, 196.0, 261.63, 220.0, 174.61, 196.0, 261.63]
        harmony.forEach((freq, index) => {
          createTone(freq, noteDuration, currentTime + index * noteDuration)
        })
      }

      // Start playing
      playMelody()
      setMusicPlaying(true)

      // Set up loop
      const musicInterval = setInterval(() => {
        if (musicEnabled && musicPlaying) {
          playMelody()
        }
      }, 6400) // 8 notes * 0.8 seconds each

      // Create mock audio object for control
      const mockAudio = {
        pause: () => {
          clearInterval(musicInterval)
          setMusicPlaying(false)
        },
        play: () => {
          setMusicPlaying(true)
          playMelody()
        },
        volume: musicVolume,
      } as HTMLAudioElement

      setBackgroundMusic(mockAudio)
    } catch (error) {
      console.log("Audio creation failed, trying alternative method:", error)

      // Fallback: Create simple beep pattern
      const playSimpleMusic = () => {
        if (!soundEnabled) return

        const beepPattern = [800, 1000, 1200, 800, 900, 700, 1000, 1200]
        beepPattern.forEach((freq, index) => {
          setTimeout(() => {
            if (musicEnabled && musicPlaying) {
              playSound(freq, 0.3, "catch")
            }
          }, index * 400)
        })
      }

      playSimpleMusic()
      setMusicPlaying(true)

      const fallbackInterval = setInterval(() => {
        if (musicEnabled && musicPlaying) {
          playSimpleMusic()
        }
      }, 3200)

      const fallbackAudio = {
        pause: () => {
          clearInterval(fallbackInterval)
          setMusicPlaying(false)
        },
        play: () => {
          setMusicPlaying(true)
          playSimpleMusic()
        },
        volume: musicVolume,
      } as HTMLAudioElement

      setBackgroundMusic(fallbackAudio)
    }
  }, [musicEnabled, musicPlaying, musicVolume, soundEnabled, playSound])

  // Create romantic background music using actual audio or uploaded file
  const createRomanticMusic = useCallback(() => {
    if (!musicEnabled || backgroundMusic) return

    // If user uploaded custom music, use that
    if (uploadedMusicUrl) {
      try {
        const audio = new Audio(uploadedMusicUrl)
        audio.loop = true
        audio.volume = musicVolume
        audio.crossOrigin = "anonymous"

        audio
          .play()
          .then(() => {
            setMusicPlaying(true)
            setBackgroundMusic(audio)
            uploadedMusicRef.current = audio
            console.log("Custom music started successfully!")
          })
          .catch((error) => {
            console.log("Custom music play failed:", error)
            // Fallback to generated music
            createGeneratedMusic()
          })

        return
      } catch (error) {
        console.log("Custom music creation failed:", error)
        // Fallback to generated music
      }
    }

    // Use generated romantic music as fallback
    createGeneratedMusic()
  }, [musicEnabled, backgroundMusic, uploadedMusicUrl, musicVolume, createGeneratedMusic])

  const stopBackgroundMusic = useCallback(() => {
    if (backgroundMusic) {
      backgroundMusic.pause()
      setBackgroundMusic(null)
    }
    if (uploadedMusicRef.current) {
      uploadedMusicRef.current.pause()
    }
    setMusicPlaying(false)
  }, [backgroundMusic])

  const toggleBackgroundMusic = useCallback(() => {
    if (musicPlaying && backgroundMusic) {
      stopBackgroundMusic()
    } else if (musicEnabled) {
      createRomanticMusic()
    }
  }, [musicPlaying, backgroundMusic, stopBackgroundMusic, createRomanticMusic, musicEnabled])

  // INTEGRATED VICTORY ANIMATION SYSTEM
  const createAnimatedElement = useCallback(
    (x: number, y: number, type: AnimatedElement["type"], options: Partial<AnimatedElement> = {}): AnimatedElement => {
      return {
        id: animationElementId.current++,
        x,
        y,
        vx: options.vx || (Math.random() - 0.5) * 4,
        vy: options.vy || (Math.random() - 0.5) * 4,
        rotation: options.rotation || Math.random() * Math.PI * 2,
        rotationSpeed: options.rotationSpeed || (Math.random() - 0.5) * 0.2,
        scale: options.scale || 0.5 + Math.random() * 0.5,
        scaleSpeed: options.scaleSpeed || (Math.random() - 0.5) * 0.02,
        opacity: options.opacity || 1,
        life: options.life || 120,
        maxLife: options.maxLife || 120,
        type,
        color: options.color || "#FF69B4",
        text: options.text,
        ...options,
      }
    },
    [],
  )

  const spawnAnimationElements = useCallback(
    (phase: number) => {
      const newElements: AnimatedElement[] = []

      switch (phase) {
        case 0: // Heart explosion phase
          for (let i = 0; i < 20; i++) {
            newElements.push(
              createAnimatedElement(
                CANVAS_WIDTH / 2 + (Math.random() - 0.5) * 100,
                CANVAS_HEIGHT / 2 + (Math.random() - 0.5) * 100,
                "heart",
                {
                  vx: (Math.random() - 0.5) * 8,
                  vy: (Math.random() - 0.5) * 8 - 2,
                  color: ["#FF69B4", "#FF1493", "#FFB6C1", "#FF4757"][Math.floor(Math.random() * 4)],
                  life: 180,
                  maxLife: 180,
                },
              ),
            )
          }
          break

        case 1: // Sparkle celebration
          for (let i = 0; i < 30; i++) {
            newElements.push(
              createAnimatedElement(Math.random() * CANVAS_WIDTH, Math.random() * CANVAS_HEIGHT, "sparkle", {
                vx: (Math.random() - 0.5) * 2,
                vy: (Math.random() - 0.5) * 2,
                color: ["#FFD700", "#FFA500", "#FFFF00", "#FF69B4"][Math.floor(Math.random() * 4)],
                life: 150,
                maxLife: 150,
                scaleSpeed: 0.01,
              }),
            )
          }
          break

        case 2: // Floating text and rings
          newElements.push(
            createAnimatedElement(CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2 - 50, "text", {
              text: "TRUE LOVE!",
              vx: 0,
              vy: -0.5,
              color: "#FF1493",
              scale: 1.5,
              life: 200,
              maxLife: 200,
            }),
          )

          for (let i = 0; i < 5; i++) {
            newElements.push(
              createAnimatedElement(CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2, "ring", {
                scale: 0.1 + i * 0.3,
                scaleSpeed: 0.02,
                color: `hsl(${320 + i * 10}, 70%, 60%)`,
                life: 160,
                maxLife: 160,
                opacity: 0.7 - i * 0.1,
              }),
            )
          }
          break

        case 3: // Final star burst
          for (let i = 0; i < 15; i++) {
            const angle = (Math.PI * 2 * i) / 15
            newElements.push(
              createAnimatedElement(CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2, "star", {
                vx: Math.cos(angle) * 6,
                vy: Math.sin(angle) * 6,
                color: ["#FFD700", "#FF69B4", "#FF1493", "#FFA500"][Math.floor(Math.random() * 4)],
                life: 120,
                maxLife: 120,
                rotationSpeed: 0.3,
              }),
            )
          }
          break
      }

      setAnimatedElements((prev) => [...prev, ...newElements])
    },
    [createAnimatedElement],
  )

  const updateAnimatedElements = useCallback(() => {
    setAnimatedElements((prev) =>
      prev
        .map((element) => ({
          ...element,
          x: element.x + element.vx,
          y: element.y + element.vy,
          rotation: element.rotation + element.rotationSpeed,
          scale: Math.max(0.1, element.scale + element.scaleSpeed),
          opacity: element.opacity * (element.life / element.maxLife),
          life: element.life - 1,
          vy: element.type === "heart" ? element.vy + 0.1 : element.vy, // gravity for hearts
        }))
        .filter((element) => element.life > 0),
    )
  }, [])

  const renderAnimatedElement = useCallback((ctx: CanvasRenderingContext2D, element: AnimatedElement) => {
    ctx.save()
    ctx.globalAlpha = element.opacity
    ctx.translate(element.x, element.y)
    ctx.rotate(element.rotation)
    ctx.scale(element.scale, element.scale)

    switch (element.type) {
      case "heart":
        ctx.fillStyle = element.color
        ctx.shadowColor = element.color
        ctx.shadowBlur = 15
        ctx.beginPath()
        ctx.moveTo(0, 15)
        ctx.bezierCurveTo(-25, -10, -45, -10, -25, 15)
        ctx.bezierCurveTo(-25, 15, -15, 25, 0, 35)
        ctx.bezierCurveTo(15, 25, 25, 15, 25, 15)
        ctx.bezierCurveTo(45, -10, 25, -10, 0, 15)
        ctx.closePath()
        ctx.fill()
        break

      case "sparkle":
        ctx.fillStyle = element.color
        ctx.shadowColor = element.color
        ctx.shadowBlur = 10
        ctx.beginPath()
        ctx.moveTo(0, -20)
        ctx.lineTo(5, -5)
        ctx.lineTo(20, 0)
        ctx.lineTo(5, 5)
        ctx.lineTo(0, 20)
        ctx.lineTo(-5, 5)
        ctx.lineTo(-20, 0)
        ctx.lineTo(-5, -5)
        ctx.closePath()
        ctx.fill()
        break

      case "star":
        ctx.fillStyle = element.color
        ctx.shadowColor = element.color
        ctx.shadowBlur = 12
        ctx.beginPath()
        for (let i = 0; i < 5; i++) {
          const angle = (i * Math.PI * 2) / 5 - Math.PI / 2
          const x = Math.cos(angle) * 15
          const y = Math.sin(angle) * 15
          if (i === 0) ctx.moveTo(x, y)
          else ctx.lineTo(x, y)

          const innerAngle = ((i + 0.5) * Math.PI * 2) / 5 - Math.PI / 2
          const innerX = Math.cos(innerAngle) * 7
          const innerY = Math.sin(innerAngle) * 7
          ctx.lineTo(innerX, innerY)
        }
        ctx.closePath()
        ctx.fill()
        break

      case "ring":
        ctx.strokeStyle = element.color
        ctx.lineWidth = 3
        ctx.shadowColor = element.color
        ctx.shadowBlur = 8
        ctx.beginPath()
        ctx.arc(0, 0, 50, 0, Math.PI * 2)
        ctx.stroke()
        break

      case "text":
        if (element.text) {
          ctx.fillStyle = element.color
          ctx.font = "bold 24px serif"
          ctx.textAlign = "center"
          ctx.shadowColor = element.color
          ctx.shadowBlur = 10
          ctx.fillText(element.text, 0, 0)
        }
        break
    }

    ctx.restore()
  }, [])

  const startVictoryAnimation = useCallback(() => {
    setGameState("animation")
    setAnimationPhase(0)
    setAnimationStartTime(Date.now())
    setAnimatedElements([])
    spawnAnimationElements(0)
    playVictorySound()
  }, [spawnAnimationElements, playVictorySound])

  // CREATE FIREWORKS
  const createFirework = useCallback((x: number, y: number) => {
    const colors = ["#FF4757", "#FF6B7A", "#FFD700", "#FF69B4", "#FFA500", "#FF1493"]
    const fireworkParticles = []

    for (let i = 0; i < 15; i++) {
      const angle = (Math.PI * 2 * i) / 15
      const speed = 2 + Math.random() * 3
      fireworkParticles.push({
        x,
        y,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        life: 60,
        maxLife: 60,
        color: colors[Math.floor(Math.random() * colors.length)],
      })
    }

    const newFirework: Firework = {
      id: fireworkIdCounter.current++,
      x,
      y,
      particles: fireworkParticles,
    }

    setFireworks((prev) => [...prev, newFirework])
  }, [])

  // TRIGGER VICTORY ANIMATION
  const triggerVictoryAnimation = useCallback(() => {
    startVictoryAnimation()
  }, [startVictoryAnimation])

  // VIDEO UPLOAD HANDLERS
  const handleVideoUpload = useCallback((event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0]
    if (!file) return

    // Validate file type
    if (!file.type.startsWith("video/")) {
      alert("Please select a valid video file")
      return
    }

    // Validate file size (max 50MB)
    if (file.size > 50 * 1024 * 1024) {
      alert("Video file is too large. Please select a file under 50MB.")
      return
    }

    setVideoUploading(true)

    // Create object URL for the video
    const videoUrl = URL.createObjectURL(file)
    setUploadedVideoUrl(videoUrl)
    setVideoUploading(false)
    setShowVideoUpload(false)

    // Show success message
    setTimeout(() => {
      alert("Video uploaded successfully! 💕 It will play when you reach 50 hearts.")
    }, 500)
  }, [])

  const handleRemoveVideo = useCallback(() => {
    if (uploadedVideoUrl) {
      URL.revokeObjectURL(uploadedVideoUrl)
      setUploadedVideoUrl(null)
    }
  }, [uploadedVideoUrl])

  const triggerVideoUpload = useCallback(() => {
    fileInputRef.current?.click()
  }, [])

  // CHARACTER UPLOAD HANDLERS
  const handleCharacterUpload = useCallback((event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0]
    if (!file) return

    // Validate file type
    if (!file.type.startsWith("image/")) {
      alert("Please select a valid image file")
      return
    }

    // Validate file size (max 10MB)
    if (file.size > 10 * 1024 * 1024) {
      alert("Image file is too large. Please select a file under 10MB.")
      return
    }

    setCharacterUploading(true)

    // Create object URL for the image
    const imageUrl = URL.createObjectURL(file)
    setUploadedCharacterUrl(imageUrl)
    setCharacterUploading(false)

    // Reset image loaded state to reload with new image
    setImageLoaded(false)

    // Show success message
    setTimeout(() => {
      alert("Character image uploaded successfully! 💕 Your custom character will appear in the game.")
    }, 500)
  }, [])

  const handleRemoveCharacter = useCallback(() => {
    if (uploadedCharacterUrl) {
      URL.revokeObjectURL(uploadedCharacterUrl)
      setUploadedCharacterUrl(null)
      setImageLoaded(false) // Reset to reload default image
    }
  }, [uploadedCharacterUrl])

  const triggerCharacterUpload = useCallback(() => {
    characterFileInputRef.current?.click()
  }, [])

  // MUSIC UPLOAD HANDLERS
  const handleMusicUpload = useCallback(
    (event: React.ChangeEvent<HTMLInputElement>) => {
      const file = event.target.files?.[0]
      if (!file) return

      // Validate file type
      if (!file.type.startsWith("audio/")) {
        alert("Please select a valid audio file (MP3, WAV, OGG)")
        return
      }

      // Validate file size (max 20MB)
      if (file.size > 20 * 1024 * 1024) {
        alert("Audio file is too large. Please select a file under 20MB.")
        return
      }

      setMusicUploading(true)

      // Create object URL for the audio
      const audioUrl = URL.createObjectURL(file)
      setUploadedMusicUrl(audioUrl)
      setMusicUploading(false)

      // Stop current music
      if (backgroundMusic) {
        backgroundMusic.pause()
        setMusicPlaying(false)
      }

      // Show success message
      setTimeout(() => {
        alert("Background music uploaded successfully! 🎵 It will play during the game.")
      }, 500)
    },
    [backgroundMusic],
  )

  const handleRemoveMusic = useCallback(() => {
    if (uploadedMusicUrl) {
      URL.revokeObjectURL(uploadedMusicUrl)
      setUploadedMusicUrl(null)
    }
    if (uploadedMusicRef.current) {
      uploadedMusicRef.current.pause()
      uploadedMusicRef.current = null
    }
    // Stop current music
    if (backgroundMusic) {
      backgroundMusic.pause()
      setMusicPlaying(false)
      setBackgroundMusic(null)
    }
  }, [uploadedMusicUrl, backgroundMusic])

  const triggerMusicUpload = useCallback(() => {
    musicFileInputRef.current?.click()
  }, [])

  const initGame = useCallback(() => {
    setScore(0)
    setHearts([])
    setParticles([])
    setFireworks([])
    setAnimatedElements([])
    setGirlPosition(50)
    setShowLoveMessage(false)
    setShowVictoryMessage(false)
    setAnimationTime(0)
    setGirlAnimationFrame(0)
    setBackgroundOffset(0)
    setVictoryAnimationTime(0)
    setGirlCelebrationFrame(0)
    setShowVideo(false)
    setVideoEnded(false)
    setShowFinalMessage(false)
    setHeartBurstActive(false)
    setAnimationPhase(0)
    setAnimationStartTime(0)
    gameSpeed.current = 1
    lastHeartSpawn.current = 0
    heartIdCounter.current = 0
    particleIdCounter.current = 0
    fireworkIdCounter.current = 0
    animationElementId.current = 0
    setShowAudioPrompt(false)
    setVideoMuted(false)
    setImageLoaded(false)

    // Start romantic background music when game starts
    if (musicEnabled && !musicPlaying) {
      setTimeout(() => createRomanticMusic(), 500)
    }
  }, [musicEnabled, musicPlaying, createRomanticMusic])

  const spawnHeart = useCallback(() => {
    const newHeart: Heart = {
      id: heartIdCounter.current++,
      x: Math.random() * (CANVAS_WIDTH - 50) + 25,
      y: -40,
      speed: 0.8 + Math.random() * 0.7,
      rotation: Math.random() * Math.PI * 2,
      scale: 0.8 + Math.random() * 0.4,
      pulse: Math.random() * Math.PI * 2,
    }
    setHearts((prev) => [...prev, newHeart])
  }, [])

  const createParticles = useCallback((x: number, y: number) => {
    const newParticles: Particle[] = []
    for (let i = 0; i < 8; i++) {
      newParticles.push({
        id: particleIdCounter.current++,
        x,
        y,
        vx: (Math.random() - 0.5) * 6,
        vy: (Math.random() - 0.5) * 6 - 3,
        life: 40,
        maxLife: 40,
        rotation: Math.random() * Math.PI * 2,
        scale: 0.5 + Math.random() * 0.5,
      })
    }
    setParticles((prev) => [...prev, ...newParticles])
  }, [])

  const checkCollision = useCallback(
    (heart: Heart) => {
      const girlX = (girlPosition / 100) * CANVAS_WIDTH
      const girlY = CANVAS_HEIGHT - 120
      return heart.x + 20 > girlX - 40 && heart.x < girlX + 40 && heart.y + 20 > girlY && heart.y < girlY + 80
    },
    [girlPosition],
  )

  const gameLoop = useCallback(() => {
    if (gameState === "animation") {
      // INTEGRATED VICTORY ANIMATION LOOP
      const currentTime = Date.now()
      const elapsed = currentTime - animationStartTime

      // Update animation phases
      if (elapsed > 1000 && animationPhase === 0) {
        setAnimationPhase(1)
        spawnAnimationElements(1)
      } else if (elapsed > 2500 && animationPhase === 1) {
        setAnimationPhase(2)
        spawnAnimationElements(2)
      } else if (elapsed > 4000 && animationPhase === 2) {
        setAnimationPhase(3)
        spawnAnimationElements(3)
      } else if (elapsed > 6000 && animationPhase === 3) {
        // Animation complete, show video or final message
        if (uploadedVideoUrl) {
          setGameState("victory")
          setShowVideo(true)
        } else {
          setShowFinalMessage(true)
        }
        return
      }

      updateAnimatedElements()
      setVictoryAnimationTime((prev) => prev + 0.1)
      setGirlCelebrationFrame((prev) => prev + 0.3)

      animationRef.current = requestAnimationFrame(gameLoop)
      return
    }

    if (gameState === "victory") {
      // VICTORY ANIMATION LOOP
      setVictoryAnimationTime((prev) => prev + 0.1)
      setGirlCelebrationFrame((prev) => prev + 0.3)

      // Update fireworks
      setFireworks((prev) =>
        prev
          .map((firework) => ({
            ...firework,
            particles: firework.particles
              .map((particle) => ({
                ...particle,
                x: particle.x + particle.vx,
                y: particle.y + particle.vy,
                vy: particle.vy + 0.1, // gravity
                life: particle.life - 1,
              }))
              .filter((particle) => particle.life > 0),
          }))
          .filter((firework) => firework.particles.length > 0),
      )

      animationRef.current = requestAnimationFrame(gameLoop)
      return
    }

    if (gameState !== "playing") return

    const now = Date.now()

    // Update animation time
    setAnimationTime((prev) => prev + 0.1)
    setBackgroundOffset((prev) => (prev + 0.5) % 100)

    // Update girl animation
    if (isMovingLeft || isMovingRight) {
      setGirlAnimationFrame((prev) => (prev + 0.2) % 4)
    }

    if (now - lastHeartSpawn.current > 2500 / gameSpeed.current) {
      spawnHeart()
      lastHeartSpawn.current = now
    }

    setHearts((prev) => {
      const updated = prev
        .map((heart) => ({
          ...heart,
          y: heart.y + heart.speed,
          rotation: heart.rotation + 0.05,
          pulse: heart.pulse + 0.1,
        }))
        .filter((heart) => heart.y < CANVAS_HEIGHT + 50)

      const remaining: Heart[] = []
      updated.forEach((heart) => {
        if (checkCollision(heart)) {
          playCatchSound()
          setScore((prevScore) => {
            const newScore = prevScore + 1

            // CHECK FOR VICTORY CONDITION
            if (newScore >= targetScore) {
              setTimeout(() => triggerVictoryAnimation(), 100)
              return newScore
            }

            // Regular love message every 10 hearts
            if (newScore % 10 === 0 && loveMessages.length > 0) {
              const messageIndex = Math.floor((newScore / 10 - 1) % loveMessages.length)
              setCurrentMessage(loveMessages[messageIndex])
              setShowLoveMessage(true)
              setTimeout(() => {
                playMessageSound()
              }, 200)
              setTimeout(() => setShowLoveMessage(false), 3000)
            }

            return newScore
          })
          createParticles(heart.x, heart.y)
        } else {
          remaining.push(heart)
        }
      })

      return remaining
    })

    setParticles((prev) =>
      prev
        .map((particle) => ({
          ...particle,
          x: particle.x + particle.vx,
          y: particle.y + particle.vy,
          vy: particle.vy + 0.1,
          life: particle.life - 1,
          rotation: particle.rotation + 0.2,
          scale: particle.scale * 0.98,
        }))
        .filter((particle) => particle.life > 0),
    )

    if (isMovingLeft) {
      setGirlPosition((prev) => Math.max(15, prev - 0.8))
    }
    if (isMovingRight) {
      setGirlPosition((prev) => Math.min(85, prev + 0.8))
    }

    gameSpeed.current = Math.min(2, 1 + score * 0.005)

    animationRef.current = requestAnimationFrame(gameLoop)
  }, [
    gameState,
    score,
    spawnHeart,
    checkCollision,
    createParticles,
    isMovingLeft,
    isMovingRight,
    loveMessages,
    playCatchSound,
    playMessageSound,
    triggerVictoryAnimation,
    animationStartTime,
    animationPhase,
    spawnAnimationElements,
    updateAnimatedElements,
    uploadedVideoUrl,
    targetScore,
  ])

  const cleanupMusic = useCallback(() => {
    if (backgroundMusic) {
      backgroundMusic.pause()
    }
  }, [backgroundMusic])

  useEffect(() => {
    if (gameState === "playing" || gameState === "victory" || gameState === "animation") {
      animationRef.current = requestAnimationFrame(gameLoop)
    } else {
      if (animationRef.current) {
        cancelAnimationFrame(animationRef.current)
      }
    }

    return () => {
      if (animationRef.current) {
        cancelAnimationFrame(animationRef.current)
      }
    }
  }, [gameState, gameLoop])

  // CANVAS RENDERING
  useEffect(() => {
    const canvas = canvasRef.current
    if (!canvas) return

    const ctx = canvas.getContext("2d")
    if (!ctx) return

    ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT)

    // Background gradient
    let gradient
    if (gameState === "victory" || gameState === "animation") {
      // Rainbow gradient for victory
      gradient = ctx.createLinearGradient(0, 0, 0, CANVAS_HEIGHT)
      gradient.addColorStop(0, "#FFE4E6")
      gradient.addColorStop(0.3, "#FFD700")
      gradient.addColorStop(0.6, "#FF69B4")
      gradient.addColorStop(1, "#FFCDD2")
    } else {
      gradient = ctx.createLinearGradient(0, 0, 0, CANVAS_HEIGHT)
      gradient.addColorStop(0, "#FFE4E6")
      gradient.addColorStop(1, "#FFCDD2")
    }
    ctx.fillStyle = gradient
    ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT)

    // Animated background hearts
    ctx.save()
    ctx.globalAlpha = gameState === "victory" || gameState === "animation" ? 0.15 : 0.08
    for (let i = 0; i < 5; i++) {
      for (let j = 0; j < 8; j++) {
        const x = (i * 80 + (j % 2) * 40) % CANVAS_WIDTH
        const y = (j * 90 + 50 + backgroundOffset) % (CANVAS_HEIGHT + 100)
        const scale = 1 + Math.sin(animationTime + i + j) * 0.1
        ctx.save()
        ctx.translate(x, y)
        ctx.scale(scale, scale)
        ctx.font = "30px serif"
        ctx.fillText("💖", -15, 15)
        ctx.restore()
      }
    }
    ctx.restore()

    // Falling hearts (only during gameplay)
    if (gameState === "playing") {
      hearts.forEach((heart) => {
        ctx.save()
        ctx.translate(heart.x, heart.y)
        ctx.rotate(heart.rotation)
        const pulseScale = heart.scale * (1 + Math.sin(heart.pulse) * 0.2)
        ctx.scale(pulseScale, pulseScale)
        ctx.shadowColor = "#FF4757"
        ctx.shadowBlur = 10
        ctx.shadowOffsetX = 0
        ctx.shadowOffsetY = 0
        ctx.fillStyle = "#FF4757"
        ctx.beginPath()
        ctx.moveTo(0, 15)
        ctx.bezierCurveTo(-25, -10, -45, -10, -25, 15)
        ctx.bezierCurveTo(-25, 15, -15, 25, 0, 35)
        ctx.bezierCurveTo(15, 25, 25, 15, 25, 15)
        ctx.bezierCurveTo(45, -10, 25, -10, 0, 15)
        ctx.closePath()
        ctx.fill()

        const heartGradient = ctx.createRadialGradient(0, 10, 0, 0, 10, 30)
        heartGradient.addColorStop(0, "#FF6B7A")
        heartGradient.addColorStop(1, "#FF4757")
        ctx.fillStyle = heartGradient
        ctx.fill()

        ctx.shadowBlur = 0
        ctx.strokeStyle = "#2C2C2C"
        ctx.lineWidth = 2.5
        ctx.stroke()
        ctx.restore()
      })
    }

    // Particles
    particles.forEach((particle) => {
      const alpha = particle.life / particle.maxLife
      ctx.save()
      ctx.globalAlpha = alpha
      ctx.translate(particle.x, particle.y)
      ctx.rotate(particle.rotation)
      ctx.scale(particle.scale, particle.scale)

      const particleType = particle.id % 3
      if (particleType === 0) {
        ctx.font = "16px serif"
        ctx.fillText("✨", -8, 8)
      } else if (particleType === 1) {
        ctx.font = "14px serif"
        ctx.fillText("💫", -7, 7)
      } else {
        ctx.font = "12px serif"
        ctx.fillText("⭐", -6, 6)
      }
      ctx.restore()
    })

    // INTEGRATED ANIMATED ELEMENTS (victory animation)
    if (gameState === "animation") {
      animatedElements.forEach((element) => {
        renderAnimatedElement(ctx, element)
      })
    }

    // FIREWORKS (victory animation)
    fireworks.forEach((firework) => {
      firework.particles.forEach((particle) => {
        const alpha = particle.life / particle.maxLife
        ctx.save()
        ctx.globalAlpha = alpha
        ctx.fillStyle = particle.color
        ctx.beginPath()
        ctx.arc(particle.x, particle.y, 3, 0, Math.PI * 2)
        ctx.fill()
        ctx.restore()
      })
    })

    // Girl character - use the provided image
    const girlX = (girlPosition / 100) * CANVAS_WIDTH
    const girlY = CANVAS_HEIGHT - 120

    // Create and load image if not already created or if character changed
    if (!girlImageRef.current || !imageLoaded) {
      girlImageRef.current = new Image()
      girlImageRef.current.crossOrigin = "anonymous"
      // Use custom character if uploaded, otherwise use default
      girlImageRef.current.src = uploadedCharacterUrl || "/girl-character.png"
      girlImageRef.current.onload = () => setImageLoaded(true)
      girlImageRef.current.onerror = () => {
        console.log("Failed to load character image")
        setImageLoaded(false)
      }
    }

    // Draw girl character image
    if (girlImageRef.current && imageLoaded) {
      ctx.save()
      if (gameState === "victory" || gameState === "animation") {
        // VICTORY CELEBRATION ANIMATION
        const celebrationBounce = Math.sin(girlCelebrationFrame * 0.5) * 10
        const celebrationSway = Math.sin(girlCelebrationFrame * 0.3) * 5
        const celebrationScale = 1 + Math.sin(victoryAnimationTime * 2) * 0.1
        ctx.translate(girlX + celebrationSway, girlY + celebrationBounce)
        ctx.scale(celebrationScale, celebrationScale)

        // Add sparkle effect around girl
        for (let i = 0; i < 8; i++) {
          const angle = (Math.PI * 2 * i) / 8 + victoryAnimationTime
          const sparkleX = Math.cos(angle) * 60
          const sparkleY = Math.sin(angle) * 40
          ctx.save()
          ctx.translate(sparkleX, sparkleY)
          ctx.font = "20px serif"
          ctx.fillText("✨", -10, 10)
          ctx.restore()
        }
      } else {
        // Normal animation
        const walkOffset = isMovingLeft || isMovingRight ? Math.sin(girlAnimationFrame) * 2 : 0
        const bounceOffset = Math.sin(animationTime * 2) * 1
        ctx.translate(girlX + walkOffset, girlY + bounceOffset)
      }

      // Scale and draw the girl character image
      const girlWidth = 80
      const girlHeight = 100
      ctx.drawImage(girlImageRef.current, -girlWidth / 2, -girlHeight / 2, girlWidth, girlHeight)
      ctx.restore()
    } else {
      // Fallback: simple placeholder while image loads
      ctx.save()
      ctx.translate(girlX, girlY)
      ctx.fillStyle = "#FF69B4"
      ctx.font = "40px serif"
      ctx.textAlign = "center"
      ctx.fillText("👧", 0, 0)
      ctx.restore()
    }
  }, [
    hearts,
    particles,
    fireworks,
    animatedElements,
    girlPosition,
    girlAnimationFrame,
    animationTime,
    backgroundOffset,
    isMovingLeft,
    isMovingRight,
    gameState,
    victoryAnimationTime,
    girlCelebrationFrame,
    imageLoaded,
    uploadedCharacterUrl,
    renderAnimatedElement,
  ])

  // EVENT HANDLERS
  const handlePointerDown = (e: React.PointerEvent) => {
    if (gameState !== "playing") return
    const rect = canvasRef.current?.getBoundingClientRect()
    if (!rect) return

    const x = e.clientX - rect.left
    const centerX = rect.width / 2

    if (x < centerX) {
      setIsMovingLeft(true)
    } else {
      setIsMovingRight(true)
    }
  }

  const handlePointerUp = () => {
    setIsMovingLeft(false)
    setIsMovingRight(false)
  }

  const addMessage = () => {
    if (newMessage.trim()) {
      setLoveMessages([...loveMessages, newMessage.trim()])
      setNewMessage("")
    }
  }

  const removeMessage = (index: number) => {
    setLoveMessages(loveMessages.filter((_, i) => i !== index))
  }

  const updateTargetScore = () => {
    const newTarget = Number.parseInt(newTargetScore)
    if (newTarget > 0 && newTarget <= 200) {
      setTargetScore(newTarget)
    }
  }

  const startGame = () => {
    // Initialize audio context with user interaction
    if (!audioInitialized) {
      try {
        const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)()
        if (audioContext.state === "suspended") {
          audioContext.resume()
        }
        setAudioInitialized(true)
        console.log("Audio initialized successfully!")
      } catch (error) {
        console.log("Audio initialization failed:", error)
      }
    }

    initGame()
    setGameState("playing")

    // Start background music immediately
    if (musicEnabled) {
      setTimeout(() => {
        console.log("Starting background music...")
        createRomanticMusic()
      }, 100)
    }
  }

  const pauseGame = () => {
    if (gameState === "playing") {
      setGameState("paused")
    } else {
      setGameState("playing")
    }
  }

  const resetGame = () => {
    setGameState("menu")
    initGame()
  }

  const handleVideoEnd = useCallback(() => {
    setVideoEnded(true)
    setTimeout(() => {
      setShowVideo(false)
      setShowFinalMessage(true)
    }, 500)
  }, [])

  const handleLoveYouToo = useCallback(() => {
    setHeartBurstActive(true)
    // Create multiple heart bursts
    for (let i = 0; i < 10; i++) {
      setTimeout(() => {
        createFirework(Math.random() * CANVAS_WIDTH, Math.random() * CANVAS_HEIGHT)
      }, i * 100)
    }

    setTimeout(() => {
      setHeartBurstActive(false)
      setShowFinalMessage(false)
      setVideoEnded(false)
      setGameState("menu")
      initGame()
    }, 3000)
  }, [createFirework, initGame])

  const handleReplay = useCallback(() => {
    setShowFinalMessage(false)
    setVideoEnded(false)
    setGameState("menu")
    initGame()
  }, [initGame])

  const handleEnableAudio = useCallback(() => {
    if (videoRef.current) {
      videoRef.current.muted = false
      videoRef.current.play()
      setShowAudioPrompt(false)
      setVideoMuted(false)
    }
  }, [])

  const toggleVideoAudio = useCallback(() => {
    if (videoRef.current) {
      videoRef.current.muted = !videoRef.current.muted
      setVideoMuted(videoRef.current.muted)
    }
  }, [])

  return (
    <div className="flex items-center justify-center min-h-screen bg-gray-200 p-4">
      <div className="bg-black rounded-[40px] p-3 shadow-2xl">
        <div className="bg-white rounded-[30px] overflow-hidden relative" style={{ width: "350px", height: "700px" }}>
          <canvas
            ref={canvasRef}
            width={CANVAS_WIDTH}
            height={CANVAS_HEIGHT}
            className="w-full h-full cursor-pointer"
            onPointerDown={handlePointerDown}
            onPointerUp={handlePointerUp}
            onPointerLeave={handlePointerUp}
          />

          {/* MENU SCREEN */}
          {gameState === "menu" && (
            <div className="absolute inset-0 bg-pink-200/40 backdrop-blur-sm flex flex-col items-center justify-center">
              <div className="text-center text-pink-800 mb-8">
                <h2 className="text-3xl font-bold mb-4">💕 Heart Catcher 💕</h2>
                <p className="text-base mb-2">Touch left/right to move</p>
                <p className="text-base">Collect {targetScore} hearts to win!</p>
              </div>
              <div className="flex flex-col gap-4">
                <Button
                  onClick={startGame}
                  className="bg-pink-500 hover:bg-pink-600 text-white px-10 py-3 rounded-full text-lg"
                >
                  <Play className="w-5 h-5 mr-2" />
                  Start Game
                </Button>
                <Button
                  onClick={() => setGameState("settings")}
                  variant="outline"
                  className="border-pink-400 text-pink-700 bg-white/90 px-10 py-3 rounded-full"
                >
                  <Settings className="w-4 h-4 mr-2" />
                  Settings
                </Button>
              </div>
            </div>
          )}

          {/* VICTORY SCREEN */}
          {gameState === "victory" && (
            <div className="absolute inset-0 bg-gradient-to-br from-pink-200/60 via-yellow-200/60 to-purple-200/60 backdrop-blur-sm flex flex-col items-center justify-center">
              <div className="text-center text-pink-800 mb-8 animate-bounce">
                <h2 className="text-4xl font-bold mb-4">🎉 VICTORY! 🎉</h2>
                <p className="text-xl mb-2">You collected all {targetScore} hearts!</p>
                <p className="text-lg">💕 TRUE LOVE ACHIEVED 💕</p>
              </div>
              <div className="flex flex-col gap-4">
                <Button
                  onClick={startGame}
                  className="bg-gradient-to-r from-pink-500 to-purple-500 hover:from-pink-600 hover:to-purple-600 text-white px-10 py-3 rounded-full text-lg"
                >
                  <Play className="w-5 h-5 mr-2" />
                  Play Again
                </Button>
                <Button
                  onClick={resetGame}
                  variant="outline"
                  className="border-pink-400 text-pink-700 bg-white/90 px-10 py-3 rounded-full"
                >
                  Back to Menu
                </Button>
              </div>
            </div>
          )}

          {/* PAUSED SCREEN */}
          {gameState === "paused" && (
            <div className="absolute inset-0 bg-pink-200/40 backdrop-blur-sm flex items-center justify-center">
              <div className="text-center">
                <h2 className="text-2xl font-bold text-pink-800 mb-6">Game Paused</h2>
                <Button onClick={pauseGame} className="bg-pink-500 hover:bg-pink-600 text-white px-6 py-3 rounded-full">
                  <Play className="w-4 h-4 mr-2" />
                  Resume
                </Button>
              </div>
            </div>
          )}

          {/* SETTINGS SCREEN */}
          {gameState === "settings" && (
            <div className="absolute inset-0 bg-white p-4 overflow-y-auto">
              <div className="max-w-sm mx-auto">
                <h2 className="text-2xl font-bold text-pink-700 mb-6 text-center">Game Settings</h2>

                {/* VIDEO UPLOAD SECTION */}
                <div className="mb-6 p-4 bg-gradient-to-r from-pink-50 to-purple-50 rounded-lg border border-pink-200">
                  <Label className="text-pink-600 font-medium mb-3 block">🎥 Custom Victory Video</Label>

                  {!uploadedVideoUrl ? (
                    <div className="text-center">
                      <p className="text-sm text-gray-600 mb-3">
                        Upload your own romantic video to play after the victory animation! 💕
                      </p>
                      <Button
                        onClick={triggerVideoUpload}
                        disabled={videoUploading}
                        className="bg-gradient-to-r from-pink-500 to-purple-500 hover:from-pink-600 hover:to-purple-600 text-white px-6 py-2 rounded-full"
                      >
                        {videoUploading ? (
                          <>
                            <div className="animate-spin rounded-full h-4 w-4 border-b-2 border-white mr-2"></div>
                            Uploading...
                          </>
                        ) : (
                          <>📁 Upload Video</>
                        )}
                      </Button>
                      <p className="text-xs text-gray-500 mt-2">Max 50MB • MP4, MOV, AVI supported</p>
                    </div>
                  ) : (
                    <div className="text-center">
                      <div className="bg-green-50 border border-green-200 rounded-lg p-3 mb-3">
                        <p className="text-green-700 font-medium">✅ Custom video uploaded!</p>
                        <p className="text-sm text-green-600">Your video will play after the victory animation</p>
                      </div>
                      <div className="flex gap-2 justify-center">
                        <Button
                          onClick={triggerVideoUpload}
                          variant="outline"
                          size="sm"
                          className="border-pink-300 text-pink-600 bg-transparent"
                        >
                          🔄 Replace
                        </Button>
                        <Button
                          onClick={handleRemoveVideo}
                          variant="outline"
                          size="sm"
                          className="border-red-300 text-red-600 hover:bg-red-50 bg-transparent"
                        >
                          🗑️ Remove
                        </Button>
                      </div>
                    </div>
                  )}

                  <input
                    ref={fileInputRef}
                    type="file"
                    accept="video/*"
                    onChange={handleVideoUpload}
                    className="hidden"
                  />
                </div>

                {/* CHARACTER CUSTOMIZATION SECTION */}
                <div className="mb-6 p-4 bg-gradient-to-r from-purple-50 to-pink-50 rounded-lg border border-purple-200">
                  <Label className="text-purple-600 font-medium mb-3 block">👤 Custom Character</Label>

                  {!uploadedCharacterUrl ? (
                    <div className="text-center">
                      <p className="text-sm text-gray-600 mb-3">
                        Upload your own character image to play as yourself! 🎮
                      </p>
                      <Button
                        onClick={triggerCharacterUpload}
                        disabled={characterUploading}
                        className="bg-gradient-to-r from-purple-500 to-pink-500 hover:from-purple-600 hover:to-pink-600 text-white px-6 py-2 rounded-full"
                      >
                        {characterUploading ? (
                          <>
                            <div className="animate-spin rounded-full h-4 w-4 border-b-2 border-white mr-2"></div>
                            Uploading...
                          </>
                        ) : (
                          <>🖼️ Upload Character</>
                        )}
                      </Button>
                      <p className="text-xs text-gray-500 mt-2">Max 10MB • PNG, JPG, GIF supported</p>
                      <p className="text-xs text-purple-600 mt-1">💡 Tip: Square images work best!</p>
                    </div>
                  ) : (
                    <div className="text-center">
                      <div className="bg-green-50 border border-green-200 rounded-lg p-3 mb-3">
                        <p className="text-green-700 font-medium">✅ Custom character uploaded!</p>
                        <p className="text-sm text-green-600">Your character is ready to catch hearts</p>
                      </div>
                      <div className="flex gap-2 justify-center">
                        <Button
                          onClick={triggerCharacterUpload}
                          variant="outline"
                          size="sm"
                          className="border-purple-300 text-purple-600 bg-transparent"
                        >
                          🔄 Replace
                        </Button>
                        <Button
                          onClick={handleRemoveCharacter}
                          variant="outline"
                          size="sm"
                          className="border-red-300 text-red-600 hover:bg-red-50 bg-transparent"
                        >
                          🗑️ Remove
                        </Button>
                      </div>
                    </div>
                  )}

                  <input
                    ref={characterFileInputRef}
                    type="file"
                    accept="image/*"
                    onChange={handleCharacterUpload}
                    className="hidden"
                  />
                </div>

                {/* MUSIC UPLOAD SECTION */}
                <div className="mb-6 p-4 bg-gradient-to-r from-green-50 to-blue-50 rounded-lg border border-green-200">
                  <Label className="text-green-600 font-medium mb-3 block">🎵 Custom Background Music</Label>

                  {!uploadedMusicUrl ? (
                    <div className="text-center">
                      <p className="text-sm text-gray-600 mb-3">
                        Upload your own romantic music to play during the game! 🎶
                      </p>
                      <Button
                        onClick={triggerMusicUpload}
                        disabled={musicUploading}
                        className="bg-gradient-to-r from-green-500 to-blue-500 hover:from-green-600 hover:to-blue-600 text-white px-6 py-2 rounded-full"
                      >
                        {musicUploading ? (
                          <>
                            <div className="animate-spin rounded-full h-4 w-4 border-b-2 border-white mr-2"></div>
                            Uploading...
                          </>
                        ) : (
                          <>🎵 Upload Music</>
                        )}
                      </Button>
                      <p className="text-xs text-gray-500 mt-2">Max 20MB • MP3, WAV, OGG supported</p>
                      <p className="text-xs text-green-600 mt-1">💡 Tip: Choose a romantic, looping track!</p>
                    </div>
                  ) : (
                    <div className="text-center">
                      <div className="bg-green-50 border border-green-200 rounded-lg p-3 mb-3">
                        <p className="text-green-700 font-medium">✅ Custom music uploaded!</p>
                        <p className="text-sm text-green-600">Your music will play during the game</p>
                      </div>
                      <div className="flex gap-2 justify-center mb-3">
                        <Button
                          onClick={triggerMusicUpload}
                          variant="outline"
                          size="sm"
                          className="border-green-300 text-green-600 bg-transparent"
                        >
                          🔄 Replace
                        </Button>
                        <Button
                          onClick={handleRemoveMusic}
                          variant="outline"
                          size="sm"
                          className="border-red-300 text-red-600 hover:bg-red-50 bg-transparent"
                        >
                          🗑️ Remove
                        </Button>
                      </div>
                      {/* Music preview controls */}
                      <div className="flex gap-2 justify-center">
                        <Button
                          onClick={() => {
                            if (uploadedMusicRef.current) {
                              if (uploadedMusicRef.current.paused) {
                                uploadedMusicRef.current.play()
                              } else {
                                uploadedMusicRef.current.pause()
                              }
                            } else if (uploadedMusicUrl) {
                              const audio = new Audio(uploadedMusicUrl)
                              audio.volume = musicVolume
                              audio.play()
                              uploadedMusicRef.current = audio
                            }
                          }}
                          variant="outline"
                          size="sm"
                          className="border-blue-300 text-blue-600 bg-blue-50"
                        >
                          🎧 Preview
                        </Button>
                      </div>
                    </div>
                  )}

                  <input
                    ref={musicFileInputRef}
                    type="file"
                    accept="audio/*"
                    onChange={handleMusicUpload}
                    className="hidden"
                  />
                </div>

                {/* TARGET SCORE SETTING */}
                <div className="mb-6">
                  <Label htmlFor="targetScore" className="text-pink-600 font-medium">
                    Target Hearts to Win
                  </Label>
                  <div className="flex gap-2 mt-2">
                    <Input
                      id="targetScore"
                      type="number"
                      min="10"
                      max="200"
                      value={newTargetScore}
                      onChange={(e) => setNewTargetScore(e.target.value)}
                      className="flex-1"
                    />
                    <Button onClick={updateTargetScore} className="bg-pink-500 hover:bg-pink-600">
                      Set
                    </Button>
                  </div>
                  <p className="text-sm text-gray-600 mt-1">Current: {targetScore} hearts</p>
                </div>

                <div className="mb-6">
                  <Label htmlFor="newMessage" className="text-pink-600 font-medium">
                    Add New Love Message
                  </Label>
                  <div className="flex gap-2 mt-2">
                    <Input
                      id="newMessage"
                      value={newMessage}
                      onChange={(e) => setNewMessage(e.target.value)}
                      placeholder="Enter your love message..."
                      className="flex-1"
                      onKeyPress={(e) => e.key === "Enter" && addMessage()}
                    />
                    <Button onClick={addMessage} className="bg-pink-500 hover:bg-pink-600">
                      <Plus className="w-4 h-4" />
                    </Button>
                  </div>
                </div>

                <div className="space-y-3 mb-6">
                  {loveMessages.map((message, index) => (
                    <div key={index} className="flex items-center gap-2 p-3 bg-pink-50 rounded-lg">
                      <span className="flex-1 text-sm text-pink-700">{message}</span>
                      <Button
                        onClick={() => removeMessage(index)}
                        size="sm"
                        variant="ghost"
                        className="text-red-500 hover:text-red-700 hover:bg-red-50"
                      >
                        <Trash2 className="w-4 h-4" />
                      </Button>
                    </div>
                  ))}
                </div>

                {/* MUSIC AND VOLUME CONTROLS */}
                <div className="mb-6 p-4 bg-gradient-to-r from-purple-50 to-pink-50 rounded-lg border border-purple-200">
                  <Label className="text-purple-600 font-medium mb-3 block">🎵 Background Music</Label>

                  <div className="flex gap-2 mb-3">
                    <Button
                      onClick={toggleBackgroundMusic}
                      variant="outline"
                      className={`flex-1 ${musicPlaying ? "bg-purple-100 border-purple-300" : "bg-gray-100"}`}
                    >
                      <Music className="w-4 h-4 mr-2" />
                      Music {musicPlaying ? "On" : "Off"}
                    </Button>
                    <Button
                      onClick={() => setSoundEnabled(!soundEnabled)}
                      variant="outline"
                      className={`flex-1 ${soundEnabled ? "bg-pink-100 border-pink-300" : "bg-gray-100"}`}
                    >
                      {soundEnabled ? <Volume2 className="w-4 h-4 mr-2" /> : <VolumeX className="w-4 h-4 mr-2" />}
                      Sound {soundEnabled ? "On" : "Off"}
                    </Button>
                  </div>

                  {/* Add test button */}
                  <div className="mb-3">
                    <Button
                      onClick={() => {
                        console.log("Testing audio...")
                        if (!audioInitialized) {
                          try {
                            const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)()
                            audioContext.resume()
                            setAudioInitialized(true)
                          } catch (error) {
                            console.log("Audio test failed:", error)
                          }
                        }
                        playSound(800, 0.5, "catch")
                        setTimeout(() => createRomanticMusic(), 200)
                      }}
                      variant="outline"
                      className="w-full border-green-300 text-green-600 bg-green-50"
                    >
                      🎵 Test Audio System
                    </Button>
                  </div>

                  <div className="space-y-2">
                    <Label htmlFor="musicVolume" className="text-sm text-purple-600">
                      Music Volume
                    </Label>
                    <input
                      id="musicVolume"
                      type="range"
                      min="0"
                      max="1"
                      step="0.1"
                      value={musicVolume}
                      onChange={(e) => setMusicVolume(Number.parseFloat(e.target.value))}
                      className="w-full h-2 bg-purple-200 rounded-lg appearance-none cursor-pointer slider"
                    />
                    <div className="text-xs text-gray-500 text-center">{Math.round(musicVolume * 100)}%</div>
                  </div>
                </div>

                <Button
                  onClick={() => setGameState("menu")}
                  className="w-full bg-pink-500 hover:bg-pink-600 text-white py-3 rounded-full"
                >
                  Back to Menu
                </Button>
              </div>
            </div>
          )}

          {/* LOVE MESSAGE POPUP */}
          {showLoveMessage && (
            <div className="absolute inset-0 flex items-center justify-center pointer-events-none">
              <div className="bg-white/95 backdrop-blur-sm rounded-2xl p-6 shadow-xl border-3 border-pink-300 animate-in zoom-in-50 fade-in duration-500 mx-4">
                <div className="text-center">
                  <div className="text-3xl mb-2">💖✨💖</div>
                  <p className="text-lg font-bold text-pink-600 mb-2 whitespace-pre-line">{currentMessage}</p>
                  <div className="text-2xl">💕💫💕</div>
                </div>
              </div>
            </div>
          )}

          {/* GAME UI */}
          {gameState === "playing" && (
            <div className="absolute top-4 left-4 right-4 flex justify-between items-center z-20">
              <div className="bg-white/90 backdrop-blur-sm rounded-full px-4 py-2">
                <span className="text-pink-600 font-bold">
                  Score: {score}/{targetScore}
                </span>
              </div>
              <div className="flex gap-2">
                {/* Music indicator */}
                {musicEnabled && (
                  <Button
                    onClick={toggleBackgroundMusic}
                    size="sm"
                    variant="outline"
                    className={`border-purple-300 text-purple-600 bg-white/90 rounded-full ${musicPlaying ? "animate-pulse" : ""}`}
                  >
                    <Music className="w-3 h-3" />
                  </Button>
                )}
                <Button
                  onClick={pauseGame}
                  size="sm"
                  variant="outline"
                  className="border-pink-300 text-pink-600 bg-white/90 rounded-full"
                >
                  <Pause className="w-3 h-3" />
                </Button>
                <Button
                  onClick={resetGame}
                  size="sm"
                  variant="outline"
                  className="border-pink-300 text-pink-600 bg-white/90 rounded-full"
                >
                  <RotateCcw className="w-3 h-3" />
                </Button>
              </div>
            </div>
          )}

          {/* FULLSCREEN VIDEO PLAYER */}
          {showVideo && (
            <div className="absolute inset-0 z-50 bg-black flex items-center justify-center animate-in fade-in duration-1000">
              <video
                ref={videoRef}
                className="w-full h-full object-cover"
                autoPlay
                playsInline
                onEnded={handleVideoEnd}
                onCanPlay={() => {
                  // Try to play with sound
                  if (videoRef.current) {
                    videoRef.current.muted = false
                    videoRef.current.play().catch(() => {
                      // If it fails, show audio enable button
                      setShowAudioPrompt(true)
                    })
                  }
                }}
                style={{ maxWidth: "100%", maxHeight: "100%" }}
              >
                {uploadedVideoUrl ? (
                  <source src={uploadedVideoUrl} type="video/mp4" />
                ) : (
                  <source src="/romantic-video.mp4" type="video/mp4" />
                )}
                Your browser does not support the video tag.
              </video>
              <div className="absolute inset-0 bg-gradient-to-t from-black/20 to-transparent pointer-events-none" />

              {/* Video info overlay */}
              {uploadedVideoUrl && (
                <div className="absolute top-4 left-4 bg-black/50 text-white px-3 py-1 rounded-full text-sm">
                  💕 Your Custom Video
                </div>
              )}

              {/* Audio enable prompt */}
              {showAudioPrompt && (
                <div className="absolute inset-0 bg-black/70 flex items-center justify-center">
                  <div className="bg-white rounded-2xl p-6 text-center max-w-sm mx-4">
                    <div className="text-4xl mb-4">🔊</div>
                    <h3 className="text-lg font-bold text-gray-800 mb-2">Enable Audio</h3>
                    <p className="text-gray-600 mb-4">Tap to play video with sound</p>
                    <button
                      onClick={handleEnableAudio}
                      className="bg-gradient-to-r from-pink-500 to-red-500 text-white px-6 py-3 rounded-full font-semibold hover:from-pink-600 hover:to-red-600 transition-all"
                    >
                      🎵 Play with Sound
                    </button>
                  </div>
                </div>
              )}

              {/* Manual audio controls */}
              <div className="absolute bottom-4 right-4 flex gap-2">
                <button
                  onClick={toggleVideoAudio}
                  className="bg-black/50 text-white p-2 rounded-full hover:bg-black/70 transition-all"
                >
                  {videoMuted ? <VolumeX className="w-4 h-4" /> : <Volume2 className="w-4 h-4" />}
                </button>
              </div>
            </div>
          )}

          {/* FINAL MESSAGE SCREEN */}
          {showFinalMessage && (
            <div className="absolute inset-0 z-50 bg-gradient-to-br from-pink-100 via-red-50 to-purple-100 flex flex-col items-center justify-center animate-in fade-in duration-1000">
              {/* Animated background hearts */}
              <div className="absolute inset-0 overflow-hidden pointer-events-none">
                {[...Array(20)].map((_, i) => (
                  <div
                    key={i}
                    className="absolute animate-bounce"
                    style={{
                      left: `${Math.random() * 100}%`,
                      top: `${Math.random() * 100}%`,
                      animationDelay: `${Math.random() * 2}s`,
                      animationDuration: `${2 + Math.random() * 2}s`,
                    }}
                  >
                    💖
                  </div>
                ))}
              </div>

              {/* Main content */}
              <div className="text-center z-10 px-6 max-w-sm">
                <div className="mb-8 animate-in slide-in-from-top duration-1000">
                  <h1 className="text-2xl font-bold text-transparent bg-gradient-to-r from-pink-600 via-red-500 to-purple-600 bg-clip-text mb-2 animate-pulse">
                    Mera Pyara Sa Bugga 💓
                  </h1>
                  <p className="text-xl font-semibold text-pink-700 animate-pulse">I love you forever</p>
                </div>

                <div className="flex flex-col gap-4 animate-in slide-in-from-bottom duration-1000 delay-500">
                  <button
                    onClick={handleLoveYouToo}
                    className="bg-gradient-to-r from-pink-500 to-red-500 hover:from-pink-600 hover:to-red-600 text-white px-8 py-4 rounded-full text-lg font-semibold shadow-lg transform hover:scale-105 transition-all duration-300 animate-pulse"
                    disabled={heartBurstActive}
                  >
                    💞 I love you too
                  </button>

                  <button
                    onClick={handleReplay}
                    className="bg-gradient-to-r from-purple-500 to-pink-500 hover:from-purple-600 hover:to-pink-600 text-white px-8 py-4 rounded-full text-lg font-semibold shadow-lg transform hover:scale-105 transition-all duration-300"
                  >
                    🔁 Replay
                  </button>
                </div>
              </div>

              {/* Heart burst effect */}
              {heartBurstActive && (
                <div className="absolute inset-0 pointer-events-none">
                  {[...Array(50)].map((_, i) => (
                    <div
                      key={i}
                      className="absolute text-2xl animate-ping"
                      style={{
                        left: `${Math.random() * 100}%`,
                        top: `${Math.random() * 100}%`,
                        animationDelay: `${Math.random() * 1}s`,
                        animationDuration: "2s",
                      }}
                    >
                      💖
                    </div>
                  ))}
                </div>
              )}
            </div>
          )}
        </div>
      </div>
    </div>
  )

  // Cleanup background music on unmount
  useEffect(() => {
    return () => {
      cleanupMusic()
      if (uploadedMusicUrl) {
        URL.revokeObjectURL(uploadedMusicUrl)
      }
      if (uploadedMusicRef.current) {
        uploadedMusicRef.current.pause()
      }
    }
  }, [cleanupMusic, uploadedMusicUrl])

  // Add this useEffect after the existing useEffects
  useEffect(() => {
    // Update volume for custom music
    if (uploadedMusicRef.current) {
      uploadedMusicRef.current.volume = musicVolume
    }

    // Restart generated music when volume changes significantly
    if (backgroundMusic && musicPlaying && musicEnabled && !uploadedMusicUrl) {
      stopBackgroundMusic()
      setTimeout(() => createRomanticMusic(), 100)
    }
  }, [
    musicVolume,
    backgroundMusic,
    musicPlaying,
    musicEnabled,
    stopBackgroundMusic,
    createRomanticMusic,
    uploadedMusicUrl,
  ])
}

